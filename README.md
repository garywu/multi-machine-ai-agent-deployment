# Multi-Machine AI Agent Deployment — Installation Scripts, Machine Protocols, and Capability Discovery

You have AI agents running in the cloud. You have local machines with GPUs, screens, and shells. The cloud can reason but cannot act. The machines can act but cannot coordinate. The missing piece is a deployment and coordination layer that registers machines, discovers capabilities, and routes work to the right node — installed with a single command.

This article covers the complete architecture for deploying a distributed AI agent platform across multiple machines, from the install script that bootstraps a new node, through the registration protocol that advertises its capabilities, to the pipeline system that chains nodes into workflows.

**What you'll learn:**

- How to design a `curl | sh` installer that detects OS/arch, installs dependencies, configures services, and registers with a coordinator
- Machine registration and heartbeat protocols — how nodes announce what they can do
- Capability discovery patterns from Consul, Kubernetes, mDNS, and UPnP
- Node/Hub/Chain architecture for building multi-step agent pipelines
- Security: key exchange, mTLS, token rotation, and the zero-trust machine identity problem
- Real-world patterns from Ray, Dask, vLLM, and Temporal for distributed AI coordination
- A complete working example with TypeScript types, protocol messages, and install scripts

---

## Table of Contents

1. [The Problem](#the-problem)
2. [Architecture Overview](#architecture-overview)
3. [Core Concepts](#core-concepts)
4. [The Install Script](#the-install-script)
5. [Machine Registration Protocol](#machine-registration-protocol)
6. [Capability Discovery](#capability-discovery)
7. [Node / Hub / Chain Architecture](#node--hub--chain-architecture)
8. [Security](#security)
9. [Patterns](#patterns)
10. [Small Examples](#small-examples)
11. [Real-World Multi-Machine AI Systems](#real-world-multi-machine-ai-systems)
12. [Comparisons](#comparisons)
13. [Anti-Patterns](#anti-patterns)
14. [References](#references)

---

## The Problem

Building a distributed AI agent platform means solving three problems simultaneously:

**1. Installation is fragmented.** You have a cloud control plane (Cloudflare Workers, Durable Objects), local machine connectors (TypeScript daemon over WebSocket), a macOS HUD app, and various worker processes. Each one has its own install procedure, its own configuration, its own auth setup. Adding a new machine means 20 minutes of manual steps that nobody documents.

**2. Machines don't know about each other.** Machine A has a GPU and can run inference. Machine B has a screen and can display status. Machine C has filesystem access and can execute code. But there is no registry, no discovery, no way for the cloud coordinator to say "I need a machine with GPU capability" and get routed to the right one.

**3. Pipelines are hardcoded.** When you need machine A to run inference, machine B to validate the output, and machine C to deploy the result, you wire it up manually. There is no abstraction for "chain these three capabilities together, routing each step to whichever machine can handle it."

These problems compound. Without automated installation, you can't easily add machines. Without capability discovery, you can't route work. Without pipeline abstractions, you can't compose multi-step workflows. The result is a system that works on one machine but collapses the moment you try to scale to two.

### What changes if you get this right

A new machine joins your agent network with a single command:

```bash
curl -sfL https://install.atlas.dev | sh -s -- \
  --role node \
  --hub https://api-mom.garywu.dev \
  --token "$ATLAS_TOKEN"
```

Within 30 seconds, the machine has:
- Detected its OS, architecture, and available resources (CPU cores, memory, GPU)
- Downloaded and installed the Atlas connector binary
- Generated a machine identity keypair
- Registered with the central coordinator, advertising its capabilities
- Started heartbeating every 60 seconds
- Configured itself as a launchd service (macOS) or systemd unit (Linux)
- Begun accepting work dispatched from the cloud

Other machines in the network immediately see the new node and can route tasks to it. The cloud coordinator's capability map updates in real time. Pipelines that were waiting for a "gpu-inference" capability suddenly have a machine to run on.

---

## Architecture Overview

The system has three layers:

```
┌─────────────────────────────────────────────────────────────────┐
│  CLOUD LAYER (Cloudflare Workers + Durable Objects)             │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ Atlas Center  │  │ Board Agents │  │ Pipeline Engine        │ │
│  │ (Registry DO) │  │ (C-suite DOs)│  │ (Workflow Orchestrator)│ │
│  │               │  │              │  │                        │ │
│  │ - machine map │  │ - perceive() │  │ - DAG execution        │ │
│  │ - capability  │  │ - reason()   │  │ - step routing         │ │
│  │   index       │  │ - dispatch() │  │ - retry/checkpoint     │ │
│  │ - presence    │  │              │  │                        │ │
│  └──────┬───────┘  └──────┬───────┘  └─────────┬──────────────┘ │
│         │                 │                     │               │
│         └─────────────┬───┘─────────────────────┘               │
│                       │                                         │
│               ┌───────┴────────┐                                │
│               │   API Mom      │  ← central proxy/registry      │
│               │ (cost routing) │                                │
│               └───────┬────────┘                                │
└───────────────────────┼─────────────────────────────────────────┘
                        │
               ═══ WebSocket ═══  (persistent, bidirectional)
                        │
┌───────────────────────┼─────────────────────────────────────────┐
│  LOCAL LAYER (per machine)                                      │
│               ┌───────┴────────┐                                │
│               │  Connector     │  ← installed by install script │
│               │                │                                │
│               │ - register()   │  → POST /v1/machines/register  │
│               │ - heartbeat()  │  → POST /v1/machines/heartbeat │
│               │ - capabilities │  → hud-notify, gpu-inference,  │
│               │ - ws handler   │    code-exec, macos-notify...  │
│               └──┬──────────┬──┘                                │
│                  │          │                                    │
│           ┌──────┴──┐  ┌───┴────────┐                           │
│           │  HUD    │  │ Workers    │                            │
│           │ (notch) │  │ (GPU, etc) │                            │
│           └─────────┘  └────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

Three machine roles:

| Role | Description | Examples |
|------|-------------|----------|
| **Node** | Worker machine. Registers capabilities, accepts dispatched tasks, returns results. | GPU inference box, macOS dev machine, headless Linux server |
| **Hub** | Coordinator. Maintains the machine registry, routes work, manages pipelines. | Cloud-hosted (Atlas Center DO), or a local machine acting as mesh coordinator |
| **Chain** | Pipeline definition. Not a machine role but a workflow spec: an ordered sequence of capabilities routed to nodes via the hub. | "inference → validate → deploy", "scrape → summarize → notify" |

---

## Core Concepts

### Machine Identity

Every machine in the network has a unique identity — a combination of a machine ID, a cryptographic keypair, and a set of metadata that describes what the machine is and what it can do.

```typescript
interface MachineIdentity {
  /** Unique machine identifier (typically hostname or UUID) */
  machineId: string;

  /** Human-readable label */
  label: string;

  /** Ed25519 public key for this machine */
  publicKey: string;

  /** Machine role in the network */
  role: "node" | "hub" | "hybrid";

  /** When this identity was created */
  createdAt: string;

  /** Network addresses where this machine can be reached */
  addresses: {
    /** Local network address (mDNS or static IP) */
    local?: string;
    /** Tailscale/WireGuard mesh address */
    mesh?: string;
    /** Public address (if directly reachable) */
    public?: string;
  };
}
```

The machine ID is generated during installation and persisted to `~/.atlas/machine-id`. It does not change across restarts, reinstalls, or network changes. The keypair is generated fresh on first install and stored in `~/.atlas/keys/`.

> **Key insight:** Machine identity must survive reboots, network changes, and even OS upgrades. Tying identity to hardware (like a MAC address) is fragile. Tying it to a hostname is ambiguous. The safest approach is a generated UUID stored on disk, paired with a keypair that proves ownership.

### Capabilities

A capability is a named, typed unit of work that a machine can perform. It has a name, an input schema, an output schema, and metadata about resource requirements.

```typescript
interface CapabilitySpec {
  /** Unique capability name (e.g., "gpu-inference", "hud-notify") */
  name: string;

  /** Human-readable description */
  description: string;

  /** Zod-compatible JSON schema for the input */
  inputSchema: Record<string, unknown>;

  /** Zod-compatible JSON schema for the output */
  outputSchema?: Record<string, unknown>;

  /** Resource requirements to execute this capability */
  resources?: {
    /** Minimum CPU cores needed */
    minCpuCores?: number;
    /** Minimum memory in GB */
    minMemoryGb?: number;
    /** Requires GPU */
    gpu?: boolean;
    /** Requires display (for HUD, notifications) */
    display?: boolean;
    /** Requires filesystem access */
    filesystem?: boolean;
    /** Requires network access */
    network?: boolean;
    /** Estimated execution time in seconds */
    estimatedDurationSeconds?: number;
  };

  /** Tags for filtering and routing */
  tags?: string[];

  /** Maximum concurrent executions on this machine */
  maxConcurrency?: number;
}
```

A minimal working example of registering capabilities:

```typescript
import * as os from "node:os";
import { z } from "zod";

const capabilities: CapabilitySpec[] = [
  {
    name: "hud-notify",
    description: "Write a message to the local HUD status queue",
    inputSchema: {
      type: "object",
      properties: {
        severity: { type: "string", enum: ["green", "yellow", "red"] },
        message: { type: "string" },
      },
      required: ["severity", "message"],
    },
    resources: { display: true },
    maxConcurrency: 10,
  },
  {
    name: "jane-status",
    description: "Return local machine status (CPU, memory, uptime)",
    inputSchema: { type: "object" },
    resources: {},
    maxConcurrency: 50,
  },
];

// Capabilities are declared statically but resource availability
// is checked dynamically at registration time
function getAvailableCapabilities(): CapabilitySpec[] {
  const hasDisplay = os.platform() === "darwin"; // macOS has a display
  return capabilities.filter((cap) => {
    if (cap.resources?.display && !hasDisplay) return false;
    if (cap.resources?.gpu && !detectGpu()) return false;
    return true;
  });
}
```

> **Key insight:** Capabilities are not the same as "what software is installed." A machine might have `ollama` installed but be out of VRAM. Capabilities must be dynamically validated at registration time and continuously re-evaluated during heartbeats. A capability that was available 60 seconds ago might not be available now.

### Registration

Registration is the process by which a machine announces itself to the hub, providing its identity, capabilities, and resource snapshot. The hub stores this in its machine registry and makes it available for capability routing.

```typescript
interface MachineRegistration {
  /** Machine identity */
  machineId: string;
  label: string;

  /** WebSocket endpoint for dispatching work */
  endpoint: string;

  /** Capabilities this machine offers */
  capabilities: CapabilitySpec[];

  /** Current resource snapshot */
  resources: ResourceSnapshot;

  /** How long this registration is valid (seconds) */
  ttlSeconds: number;

  /** Protocol version */
  protocolVersion: string;
}

interface ResourceSnapshot {
  platform: string;
  arch: string;
  cpus: number;
  memoryGb: number;
  freeMemoryGb: number;
  loadAvg: [number, number, number];
  uptimeSeconds: number;
  gpus?: GpuInfo[];
  diskFreeGb?: number;
}

interface GpuInfo {
  name: string;
  memoryMb: number;
  freeMemoryMb: number;
  utilization: number;
}
```

Registration is idempotent. Calling register with the same machine ID updates the existing registration rather than creating a duplicate. The TTL ensures that if a machine crashes without deregistering, its registration expires and it is removed from the capability index.

> **Key insight:** Registration must be idempotent and TTL-based. Machines crash. Networks partition. Graceful deregistration (DELETE on shutdown) is best-effort. The TTL is the real garbage collector. Set it to 3x your heartbeat interval — if you heartbeat every 60s, set TTL to 180s.

### Heartbeat

The heartbeat is a periodic signal from a machine to the hub, confirming liveness and updating resource state. It is lighter than a full registration — it refreshes the TTL and updates dynamic fields (free memory, load average, GPU utilization) without resending the full capability list.

```typescript
interface HeartbeatPayload {
  machineId: string;
  capabilities: string[]; // Just the names, not full specs
  resources: ResourceSnapshot;
  busy: boolean;
  currentTasks?: string[]; // IDs of tasks currently executing
  version: string;
}

// Heartbeat response from the hub
interface HeartbeatResponse {
  /** Whether the hub still recognizes this machine */
  registered: boolean;
  /** If false, machine should re-register */
  reRegister?: boolean;
  /** Pending configuration updates */
  config?: Partial<ConnectorConfig>;
  /** Commands to execute (e.g., "upgrade", "restart") */
  commands?: HubCommand[];
}
```

The heartbeat response is also a command channel. The hub can use it to tell a machine to upgrade, change its configuration, or even deregister. This avoids the need for a separate push channel for administrative commands (though the WebSocket connection provides one too).

---

## The Install Script

### Design Principles

The best install scripts in the ecosystem share common patterns. [K3s](https://k3s.io/) installs a complete Kubernetes distribution with `curl -sfL https://get.k3s.io | sh -`. [Homebrew](https://brew.sh/) bootstraps a package manager. [mise](https://mise.jdx.dev/) and [proto](https://moonrepo.dev/proto) install runtime version managers. [Tailscale](https://tailscale.com/) installs a mesh VPN.

What makes them work:

1. **Detect everything, assume nothing.** OS, architecture, init system, shell, existing installations.
2. **Single entry point, multiple roles.** The same script installs a server or an agent depending on environment variables.
3. **Idempotent.** Running the script twice produces the same result. It detects existing installations and skips or upgrades.
4. **Verifiable.** Binary checksums are validated before installation.
5. **Service integration.** The script configures the installed binary as a system service (launchd, systemd, openrc) so it survives reboots.
6. **Minimal dependencies.** Only curl/wget, tar/unzip, and a shell. Everything else is bundled.

### K3s Install Script Anatomy

The [K3s install script](https://github.com/k3s-io/k3s/blob/main/install.sh) is the gold standard for `curl | sh` distributed system installers. Its structure:

```
verify_system()          → detect init system (systemd/openrc)
setup_verify_arch()      → map uname -m to binary suffix
setup_env()              → determine role (server/agent from K3S_URL)
download_and_verify()    → fetch binary + SHA256 checksum, verify
setup_selinux()          → configure SELinux policies
create_env_file()        → write /etc/systemd/system/k3s.env
create_service_file()    → write systemd unit or openrc init script
create_killall()         → write k3s-killall.sh
create_uninstall()       → write k3s-uninstall.sh
systemd_enable_start()   → enable and start the service
```

Key decisions:
- Role is determined by a single env var (`K3S_URL` being set means "agent mode")
- Version pinning via `INSTALL_K3S_VERSION` or auto-detect latest stable
- Skip download with `INSTALL_K3S_SKIP_DOWNLOAD` for air-gapped installs
- Force restart with `INSTALL_K3S_FORCE_RESTART`

### Atlas Connector Install Script

Here is a complete install script for the Atlas connector, following the patterns from k3s and Tailscale:

```bash
#!/bin/sh
# Atlas Connector Installer
# Usage: curl -sfL https://install.atlas.dev | sh -s -- [options]
#
# Options:
#   --role <node|hub>         Machine role (default: node)
#   --hub <url>               Hub URL (required for nodes)
#   --token <token>           Registration token
#   --machine-id <id>         Machine ID (default: hostname)
#   --label <label>           Human-readable label
#   --channel <stable|edge>   Release channel (default: stable)
#   --version <version>       Pin to specific version
#   --skip-service            Don't configure system service
#   --skip-start              Don't start after install
#   --uninstall               Remove Atlas connector
#
# Environment variables:
#   ATLAS_HUB_URL             Same as --hub
#   ATLAS_TOKEN               Same as --token
#   ATLAS_ROLE                Same as --role
#   ATLAS_MACHINE_ID          Same as --machine-id
#   ATLAS_INSTALL_DIR         Install directory (default: /usr/local/bin)
#   ATLAS_CONFIG_DIR          Config directory (default: ~/.atlas)

set -eu

# ── Constants ────────────────────────────────────────────────────────────────

ATLAS_REPO="garywu/atlas"
ATLAS_API="https://api.github.com/repos/${ATLAS_REPO}/releases"
ATLAS_DEFAULT_HUB="https://api-mom.garywu.dev"
ATLAS_CONFIG_DIR="${ATLAS_CONFIG_DIR:-${HOME}/.atlas}"
ATLAS_INSTALL_DIR="${ATLAS_INSTALL_DIR:-/usr/local/bin}"
ATLAS_LOG_DIR="${ATLAS_CONFIG_DIR}/logs"

# ── Color Output ─────────────────────────────────────────────────────────────

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

info()  { printf "${BLUE}[atlas]${NC} %s\n" "$*"; }
ok()    { printf "${GREEN}[atlas]${NC} %s\n" "$*"; }
warn()  { printf "${YELLOW}[atlas]${NC} %s\n" "$*" >&2; }
fatal() { printf "${RED}[atlas]${NC} %s\n" "$*" >&2; exit 1; }

# ── OS / Architecture Detection ──────────────────────────────────────────────

detect_os() {
  OS="$(uname -s | tr '[:upper:]' '[:lower:]')"
  case "$OS" in
    darwin)  OS="darwin" ;;
    linux)   OS="linux" ;;
    *)       fatal "Unsupported OS: $OS" ;;
  esac
}

detect_arch() {
  ARCH="$(uname -m)"
  case "$ARCH" in
    x86_64|amd64)   ARCH="amd64" ;;
    aarch64|arm64)   ARCH="arm64" ;;
    armv7l|armhf)    ARCH="arm" ;;
    *)               fatal "Unsupported architecture: $ARCH" ;;
  esac
}

detect_init_system() {
  if [ -d /run/systemd/system ]; then
    INIT_SYSTEM="systemd"
  elif command -v launchctl >/dev/null 2>&1; then
    INIT_SYSTEM="launchd"
  elif command -v openrc-run >/dev/null 2>&1; then
    INIT_SYSTEM="openrc"
  else
    INIT_SYSTEM="none"
    warn "No supported init system detected — skipping service setup"
  fi
}

# ── Dependency Checks ────────────────────────────────────────────────────────

check_dependencies() {
  for cmd in curl tar; do
    if ! command -v "$cmd" >/dev/null 2>&1; then
      fatal "Required command not found: $cmd"
    fi
  done

  # Check for Node.js (required for the connector)
  if ! command -v node >/dev/null 2>&1; then
    warn "Node.js not found — attempting to install via mise"
    install_node_runtime
  fi

  NODE_VERSION="$(node --version 2>/dev/null || echo 'none')"
  info "Node.js version: $NODE_VERSION"
}

install_node_runtime() {
  if command -v mise >/dev/null 2>&1; then
    info "Installing Node.js via mise..."
    mise install node@lts
    mise use --global node@lts
  elif command -v brew >/dev/null 2>&1; then
    info "Installing Node.js via Homebrew..."
    brew install node
  elif command -v apt-get >/dev/null 2>&1; then
    info "Installing Node.js via apt..."
    curl -fsSL https://deb.nodesource.com/setup_lts.x | sh -
    apt-get install -y nodejs
  else
    fatal "Cannot install Node.js — please install it manually"
  fi
}

# ── Parse Arguments ──────────────────────────────────────────────────────────

parse_args() {
  ROLE="${ATLAS_ROLE:-node}"
  HUB_URL="${ATLAS_HUB_URL:-${ATLAS_DEFAULT_HUB}}"
  TOKEN="${ATLAS_TOKEN:-}"
  MACHINE_ID="${ATLAS_MACHINE_ID:-$(hostname)}"
  LABEL=""
  CHANNEL="stable"
  VERSION=""
  SKIP_SERVICE=false
  SKIP_START=false
  UNINSTALL=false

  while [ $# -gt 0 ]; do
    case "$1" in
      --role)        ROLE="$2";       shift 2 ;;
      --hub)         HUB_URL="$2";    shift 2 ;;
      --token)       TOKEN="$2";      shift 2 ;;
      --machine-id)  MACHINE_ID="$2"; shift 2 ;;
      --label)       LABEL="$2";      shift 2 ;;
      --channel)     CHANNEL="$2";    shift 2 ;;
      --version)     VERSION="$2";    shift 2 ;;
      --skip-service) SKIP_SERVICE=true; shift ;;
      --skip-start)  SKIP_START=true; shift ;;
      --uninstall)   UNINSTALL=true;  shift ;;
      *)             fatal "Unknown option: $1" ;;
    esac
  done

  if [ -z "$LABEL" ]; then
    LABEL="$(hostname) (${OS}/${ARCH})"
  fi
}

# ── Version Resolution ───────────────────────────────────────────────────────

resolve_version() {
  if [ -n "$VERSION" ]; then
    info "Using pinned version: $VERSION"
    return
  fi

  info "Resolving latest ${CHANNEL} version..."
  VERSION=$(curl -sfL "${ATLAS_API}/latest" | \
    grep '"tag_name"' | \
    sed -E 's/.*"tag_name": *"([^"]+)".*/\1/')

  if [ -z "$VERSION" ]; then
    fatal "Could not determine latest version"
  fi

  info "Latest version: $VERSION"
}

# ── Download and Verify ──────────────────────────────────────────────────────

download_binary() {
  BINARY_NAME="atlas-connector-${OS}-${ARCH}"
  DOWNLOAD_URL="https://github.com/${ATLAS_REPO}/releases/download/${VERSION}/${BINARY_NAME}.tar.gz"
  CHECKSUM_URL="${DOWNLOAD_URL}.sha256"

  TMPDIR="$(mktemp -d)"
  trap 'rm -rf "$TMPDIR"' EXIT

  info "Downloading ${BINARY_NAME} ${VERSION}..."
  curl -sfL -o "${TMPDIR}/connector.tar.gz" "$DOWNLOAD_URL" || \
    fatal "Download failed: $DOWNLOAD_URL"

  info "Verifying checksum..."
  curl -sfL -o "${TMPDIR}/connector.sha256" "$CHECKSUM_URL" || \
    warn "Checksum file not available — skipping verification"

  if [ -f "${TMPDIR}/connector.sha256" ]; then
    cd "$TMPDIR"
    if command -v sha256sum >/dev/null 2>&1; then
      echo "$(cat connector.sha256)  connector.tar.gz" | sha256sum -c - || \
        fatal "Checksum verification failed"
    elif command -v shasum >/dev/null 2>&1; then
      echo "$(cat connector.sha256)  connector.tar.gz" | shasum -a 256 -c - || \
        fatal "Checksum verification failed"
    fi
    ok "Checksum verified"
  fi

  info "Extracting..."
  tar xzf "${TMPDIR}/connector.tar.gz" -C "$TMPDIR"
}

# ── Install ──────────────────────────────────────────────────────────────────

install_binary() {
  info "Installing to ${ATLAS_INSTALL_DIR}/"

  # Ensure install directory exists
  if [ ! -w "$ATLAS_INSTALL_DIR" ]; then
    if command -v sudo >/dev/null 2>&1; then
      sudo mkdir -p "$ATLAS_INSTALL_DIR"
      sudo cp "${TMPDIR}/atlas-connector" "${ATLAS_INSTALL_DIR}/"
      sudo chmod +x "${ATLAS_INSTALL_DIR}/atlas-connector"
    else
      fatal "Cannot write to ${ATLAS_INSTALL_DIR} — run as root or set ATLAS_INSTALL_DIR"
    fi
  else
    mkdir -p "$ATLAS_INSTALL_DIR"
    cp "${TMPDIR}/atlas-connector" "${ATLAS_INSTALL_DIR}/"
    chmod +x "${ATLAS_INSTALL_DIR}/atlas-connector"
  fi

  ok "Installed atlas-connector to ${ATLAS_INSTALL_DIR}/atlas-connector"
}

# ── Configuration ────────────────────────────────────────────────────────────

setup_config() {
  mkdir -p "$ATLAS_CONFIG_DIR"
  mkdir -p "$ATLAS_LOG_DIR"
  mkdir -p "${ATLAS_CONFIG_DIR}/keys"

  # Generate machine ID file if not present
  MACHINE_ID_FILE="${ATLAS_CONFIG_DIR}/machine-id"
  if [ ! -f "$MACHINE_ID_FILE" ]; then
    if command -v uuidgen >/dev/null 2>&1; then
      uuidgen | tr '[:upper:]' '[:lower:]' > "$MACHINE_ID_FILE"
    else
      cat /proc/sys/kernel/random/uuid > "$MACHINE_ID_FILE" 2>/dev/null || \
        head -c 16 /dev/urandom | od -An -tx1 | tr -d ' \n' | \
        sed 's/\(.\{8\}\)\(.\{4\}\)\(.\{4\}\)\(.\{4\}\)/\1-\2-\3-\4-/' \
        > "$MACHINE_ID_FILE"
    fi
    info "Generated machine ID: $(cat "$MACHINE_ID_FILE")"
  else
    info "Existing machine ID: $(cat "$MACHINE_ID_FILE")"
  fi

  # Generate Ed25519 keypair if not present
  KEY_FILE="${ATLAS_CONFIG_DIR}/keys/machine.key"
  PUB_FILE="${ATLAS_CONFIG_DIR}/keys/machine.pub"
  if [ ! -f "$KEY_FILE" ]; then
    info "Generating Ed25519 keypair..."
    if command -v ssh-keygen >/dev/null 2>&1; then
      ssh-keygen -t ed25519 -f "$KEY_FILE" -N "" -C "atlas-connector@$(hostname)" \
        >/dev/null 2>&1
      ok "Keypair generated"
    elif command -v openssl >/dev/null 2>&1; then
      openssl genpkey -algorithm Ed25519 -out "$KEY_FILE" 2>/dev/null
      openssl pkey -in "$KEY_FILE" -pubout -out "$PUB_FILE" 2>/dev/null
      ok "Keypair generated (OpenSSL)"
    else
      warn "No key generation tool available — skipping keypair generation"
    fi
    chmod 600 "$KEY_FILE" 2>/dev/null || true
  fi

  # Write config file
  CONFIG_FILE="${ATLAS_CONFIG_DIR}/connector.json"
  cat > "$CONFIG_FILE" <<EOF
{
  "machineId": "$(cat "$MACHINE_ID_FILE")",
  "label": "${LABEL}",
  "role": "${ROLE}",
  "hubUrl": "${HUB_URL}",
  "protocolVersion": "1.0",
  "heartbeatIntervalMs": 60000,
  "registrationTtlSeconds": 180,
  "capabilities": {
    "autoDetect": true,
    "additional": []
  }
}
EOF

  ok "Configuration written to $CONFIG_FILE"
}

# ── Service Configuration ────────────────────────────────────────────────────

setup_service() {
  if [ "$SKIP_SERVICE" = true ]; then
    info "Skipping service setup (--skip-service)"
    return
  fi

  case "$INIT_SYSTEM" in
    launchd)  setup_launchd ;;
    systemd)  setup_systemd ;;
    openrc)   setup_openrc ;;
    *)        warn "Cannot configure service — no supported init system" ;;
  esac
}

setup_launchd() {
  PLIST_PATH="${HOME}/Library/LaunchAgents/com.atlas.connector.plist"
  info "Writing launchd plist: $PLIST_PATH"

  cat > "$PLIST_PATH" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.atlas.connector</string>

    <key>ProgramArguments</key>
    <array>
        <string>${ATLAS_INSTALL_DIR}/atlas-connector</string>
    </array>

    <key>WorkingDirectory</key>
    <string>${ATLAS_CONFIG_DIR}</string>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>

    <key>ThrottleInterval</key>
    <integer>10</integer>

    <key>StandardOutPath</key>
    <string>${ATLAS_LOG_DIR}/connector.log</string>

    <key>StandardErrorPath</key>
    <string>${ATLAS_LOG_DIR}/connector.err.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>ATLAS_CONFIG_DIR</key>
        <string>${ATLAS_CONFIG_DIR}</string>
        <key>ATLAS_TOKEN</key>
        <string>${TOKEN}</string>
    </dict>
</dict>
</plist>
EOF

  ok "launchd service configured"
}

setup_systemd() {
  UNIT_FILE="/etc/systemd/system/atlas-connector.service"
  info "Writing systemd unit: $UNIT_FILE"

  sudo tee "$UNIT_FILE" >/dev/null <<EOF
[Unit]
Description=Atlas Connector
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=${ATLAS_INSTALL_DIR}/atlas-connector
WorkingDirectory=${ATLAS_CONFIG_DIR}
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
Environment=ATLAS_CONFIG_DIR=${ATLAS_CONFIG_DIR}
Environment=ATLAS_TOKEN=${TOKEN}
Environment=PATH=/usr/local/bin:/usr/bin:/bin

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=${ATLAS_CONFIG_DIR}
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

  sudo systemctl daemon-reload
  sudo systemctl enable atlas-connector
  ok "systemd service configured and enabled"
}

setup_openrc() {
  INIT_SCRIPT="/etc/init.d/atlas-connector"
  info "Writing OpenRC init script: $INIT_SCRIPT"

  sudo tee "$INIT_SCRIPT" >/dev/null <<'INITEOF'
#!/sbin/openrc-run
description="Atlas Connector"
command="${ATLAS_INSTALL_DIR}/atlas-connector"
command_background=true
pidfile="/var/run/atlas-connector.pid"
output_log="${ATLAS_LOG_DIR}/connector.log"
error_log="${ATLAS_LOG_DIR}/connector.err.log"

depend() {
    need net
    after firewall
}
INITEOF

  sudo chmod +x "$INIT_SCRIPT"
  sudo rc-update add atlas-connector default
  ok "OpenRC service configured"
}

# ── Initial Registration ─────────────────────────────────────────────────────

register_with_hub() {
  if [ -z "$TOKEN" ]; then
    warn "No token provided — skipping initial registration"
    warn "Set ATLAS_TOKEN or use --token to register with the hub"
    return
  fi

  info "Registering with hub at ${HUB_URL}..."

  RESPONSE=$(curl -sf -X POST \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${TOKEN}" \
    -d "{
      \"machine_id\": \"$(cat "${ATLAS_CONFIG_DIR}/machine-id")\",
      \"label\": \"${LABEL}\",
      \"role\": \"${ROLE}\",
      \"resources\": {
        \"platform\": \"${OS}\",
        \"arch\": \"${ARCH}\",
        \"cpus\": $(nproc 2>/dev/null || sysctl -n hw.ncpu 2>/dev/null || echo 1)
      },
      \"ttl_seconds\": 180
    }" \
    "${HUB_URL}/v1/machines/register" 2>&1) || {
    warn "Registration failed — connector will retry on startup"
    return
  }

  ok "Registered with hub"
}

# ── Start Service ────────────────────────────────────────────────────────────

start_service() {
  if [ "$SKIP_START" = true ]; then
    info "Skipping service start (--skip-start)"
    return
  fi

  case "$INIT_SYSTEM" in
    launchd)
      launchctl load "$PLIST_PATH" 2>/dev/null || true
      launchctl start com.atlas.connector 2>/dev/null || true
      ok "Service started via launchd"
      ;;
    systemd)
      sudo systemctl start atlas-connector
      ok "Service started via systemd"
      ;;
    openrc)
      sudo rc-service atlas-connector start
      ok "Service started via OpenRC"
      ;;
    *)
      info "Start the connector manually:"
      info "  atlas-connector"
      ;;
  esac
}

# ── Uninstall ────────────────────────────────────────────────────────────────

do_uninstall() {
  info "Uninstalling Atlas connector..."

  # Stop service
  case "$INIT_SYSTEM" in
    launchd)
      launchctl unload "${HOME}/Library/LaunchAgents/com.atlas.connector.plist" 2>/dev/null || true
      rm -f "${HOME}/Library/LaunchAgents/com.atlas.connector.plist"
      ;;
    systemd)
      sudo systemctl stop atlas-connector 2>/dev/null || true
      sudo systemctl disable atlas-connector 2>/dev/null || true
      sudo rm -f /etc/systemd/system/atlas-connector.service
      sudo systemctl daemon-reload
      ;;
    openrc)
      sudo rc-service atlas-connector stop 2>/dev/null || true
      sudo rc-update del atlas-connector 2>/dev/null || true
      sudo rm -f /etc/init.d/atlas-connector
      ;;
  esac

  # Deregister from hub
  if [ -f "${ATLAS_CONFIG_DIR}/machine-id" ] && [ -n "${TOKEN:-}" ]; then
    MID="$(cat "${ATLAS_CONFIG_DIR}/machine-id")"
    curl -sf -X DELETE \
      -H "Authorization: Bearer ${TOKEN}" \
      "${HUB_URL}/v1/machines/${MID}" 2>/dev/null || true
    info "Deregistered from hub"
  fi

  # Remove binary
  rm -f "${ATLAS_INSTALL_DIR}/atlas-connector"

  # Ask about config
  printf "Remove configuration directory %s? [y/N] " "$ATLAS_CONFIG_DIR"
  read -r answer
  if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    rm -rf "$ATLAS_CONFIG_DIR"
    ok "Configuration removed"
  else
    info "Configuration preserved at $ATLAS_CONFIG_DIR"
  fi

  ok "Atlas connector uninstalled"
  exit 0
}

# ── Main ─────────────────────────────────────────────────────────────────────

main() {
  info "Atlas Connector Installer"
  info "========================="

  detect_os
  detect_arch
  detect_init_system
  parse_args "$@"

  info "OS: ${OS}  Arch: ${ARCH}  Init: ${INIT_SYSTEM}  Role: ${ROLE}"

  if [ "$UNINSTALL" = true ]; then
    do_uninstall
  fi

  check_dependencies
  resolve_version
  download_binary
  install_binary
  setup_config
  setup_service
  register_with_hub
  start_service

  ok "========================================="
  ok "Atlas Connector installed successfully!"
  ok "========================================="
  ok ""
  ok "  Machine ID:  $(cat "${ATLAS_CONFIG_DIR}/machine-id")"
  ok "  Role:        ${ROLE}"
  ok "  Hub:         ${HUB_URL}"
  ok "  Config:      ${ATLAS_CONFIG_DIR}/connector.json"
  ok "  Logs:        ${ATLAS_LOG_DIR}/connector.log"
  ok ""
  ok "Manage the service:"
  case "$INIT_SYSTEM" in
    launchd)
      ok "  launchctl start com.atlas.connector"
      ok "  launchctl stop com.atlas.connector"
      ok "  tail -f ${ATLAS_LOG_DIR}/connector.log"
      ;;
    systemd)
      ok "  sudo systemctl status atlas-connector"
      ok "  sudo journalctl -u atlas-connector -f"
      ;;
    openrc)
      ok "  sudo rc-service atlas-connector status"
      ok "  tail -f ${ATLAS_LOG_DIR}/connector.log"
      ;;
  esac
}

main "$@"
```

### What this script does differently from most installers

1. **Generates a persistent machine identity.** Not just installing a binary — creating a cryptographic identity that will represent this machine in the network.

2. **Auto-detects and installs Node.js** via whichever package manager is available (mise, Homebrew, apt).

3. **Configures the init system automatically.** Detects launchd (macOS), systemd (Linux), or OpenRC and writes the appropriate service configuration with security hardening.

4. **Registers with the hub** during installation. The machine is immediately discoverable by other nodes in the network.

5. **Includes a complete uninstaller** that deregisters from the hub, stops the service, removes the binary, and optionally cleans up configuration.

> **Key insight:** The best install scripts do not just copy a binary. They establish identity, configure supervision, and integrate the new component into an existing system. The install script _is_ the onboarding protocol.

---

## Machine Registration Protocol

### Protocol Messages

The connector communicates with the hub using a typed message protocol over WebSocket. Here is the complete type system:

```typescript
// ── Messages from Hub → Machine (Server Messages) ──────────────────────────

type ServerMessage =
  | DispatchRequest
  | PingMessage
  | PongMessage
  | UpdateCommand
  | ConfigPush
  | CapabilityQuery;

interface DispatchRequest {
  type: "request";
  /** Unique request ID for correlation */
  id: string;
  /** Capability to invoke */
  capability: string;
  /** Input payload for the capability handler */
  input: unknown;
  /** Optional: which pipeline this request belongs to */
  pipelineId?: string;
  /** Optional: step index within the pipeline */
  stepIndex?: number;
  /** Timeout in milliseconds */
  timeoutMs?: number;
}

interface PingMessage {
  type: "ping";
  timestamp: number;
}

interface PongMessage {
  type: "pong";
  timestamp: number;
}

interface UpdateCommand {
  type: "update";
  /** Target version to upgrade to */
  version?: string;
  /** URL to download the update from */
  url?: string;
  /** Whether to restart after update */
  restart?: boolean;
}

interface ConfigPush {
  type: "config-push";
  /** Partial config to merge */
  config: Partial<{
    heartbeatIntervalMs: number;
    capabilities: string[];
    logLevel: "debug" | "info" | "warn" | "error";
  }>;
}

interface CapabilityQuery {
  type: "capability-query";
  /** Request ID */
  id: string;
  /** Capability name to check */
  capability: string;
}

// ── Messages from Machine → Hub (Client Messages) ──────────────────────────

type ClientMessage =
  | ResponseMessage
  | StatusReport
  | PingMessage
  | PongMessage
  | CapabilityAdvertisement
  | TaskProgress;

interface ResponseMessage {
  type: "response";
  /** Correlates with the request ID */
  id: string;
  /** HTTP-style status code */
  status: number;
  /** Result payload */
  body: unknown;
}

interface StatusReport {
  type: "status-report";
  /** Machine ID */
  machine: string;
  /** Whether the machine is currently processing a task */
  busy: boolean;
  /** Connector version */
  version: string;
  /** Current resource snapshot */
  resources?: ResourceSnapshot;
  /** IDs of tasks currently executing */
  activeTasks?: string[];
}

interface CapabilityAdvertisement {
  type: "capability-advertisement";
  /** Full capability specs (sent on connect and when capabilities change) */
  capabilities: CapabilitySpec[];
}

interface TaskProgress {
  type: "task-progress";
  /** Request ID this progress is for */
  id: string;
  /** Progress percentage (0-100) */
  progress: number;
  /** Human-readable status message */
  message?: string;
  /** Partial result (for streaming) */
  partial?: unknown;
}
```

### Registration Flow

The full registration flow has six phases:

```
Machine                                Hub (Atlas Center DO)
  │                                        │
  │  1. POST /v1/machines/register         │
  │  ──────────────────────────────────►   │
  │  { machineId, capabilities,            │
  │    resources, ttlSeconds }             │
  │                                        │  Store in machine registry
  │  ◄──────────────────────────────────   │  Update capability index
  │  { registered: true, hubWsUrl }        │
  │                                        │
  │  2. WebSocket connect                  │
  │  ══════════════════════════════════►   │
  │  Authorization: Bearer <token>         │
  │                                        │
  │  3. status-report (initial)            │
  │  ──────────────────────────────────►   │
  │  { machine, busy: false, version }     │  Mark machine as "connected"
  │                                        │
  │  4. capability-advertisement           │
  │  ──────────────────────────────────►   │
  │  { capabilities: [...full specs] }     │  Merge into capability index
  │                                        │
  │  5. heartbeat loop (every 60s)         │
  │  ──────────────────────────────────►   │
  │  POST /v1/machines/heartbeat           │  Refresh TTL
  │  { machineId, resources }              │  Update resource snapshot
  │  ◄──────────────────────────────────   │
  │  { registered: true, commands: [] }    │
  │                                        │
  │  6. dispatch (when work arrives)       │
  │  ◄══════════════════════════════════   │
  │  { type: "request", capability,        │
  │    input, id }                         │
  │  ══════════════════════════════════►   │
  │  { type: "response", id, status,       │
  │    body }                              │
  │                                        │
```

### Hub-Side Registry (Durable Object)

The hub maintains a machine registry as a Durable Object. Here is the core data structure and routing logic:

```typescript
interface MachineRegistry {
  /** All registered machines, keyed by machine ID */
  machines: Map<string, RegisteredMachine>;

  /** Capability index: capability name → set of machine IDs */
  capabilityIndex: Map<string, Set<string>>;

  /** Active WebSocket connections, keyed by machine ID */
  connections: Map<string, WebSocket>;
}

interface RegisteredMachine {
  machineId: string;
  label: string;
  role: "node" | "hub" | "hybrid";
  capabilities: CapabilitySpec[];
  resources: ResourceSnapshot;
  registeredAt: number;
  lastHeartbeat: number;
  ttlSeconds: number;
  busy: boolean;
  activeTasks: string[];
  version: string;
  connected: boolean;
}

class AtlasCenterDO {
  private registry: MachineRegistry = {
    machines: new Map(),
    capabilityIndex: new Map(),
    connections: new Map(),
  };

  /**
   * Find machines that can handle a given capability.
   * Returns machines sorted by suitability (least busy, most resources).
   */
  findMachinesForCapability(
    capabilityName: string,
    requirements?: Partial<CapabilitySpec["resources"]>
  ): RegisteredMachine[] {
    const machineIds = this.registry.capabilityIndex.get(capabilityName);
    if (!machineIds) return [];

    const now = Date.now();
    const candidates: RegisteredMachine[] = [];

    for (const id of machineIds) {
      const machine = this.registry.machines.get(id);
      if (!machine) continue;

      // Check TTL — skip expired registrations
      const expiresAt = machine.lastHeartbeat + machine.ttlSeconds * 1000;
      if (now > expiresAt) {
        this.deregisterMachine(id);
        continue;
      }

      // Check if machine is connected
      if (!machine.connected) continue;

      // Check resource requirements
      if (requirements) {
        if (requirements.gpu && !machine.resources.gpus?.length) continue;
        if (
          requirements.minMemoryGb &&
          machine.resources.freeMemoryGb < requirements.minMemoryGb
        )
          continue;
        if (
          requirements.minCpuCores &&
          machine.resources.cpus < requirements.minCpuCores
        )
          continue;
      }

      // Check concurrency limits
      const capSpec = machine.capabilities.find(
        (c) => c.name === capabilityName
      );
      if (
        capSpec?.maxConcurrency &&
        machine.activeTasks.length >= capSpec.maxConcurrency
      )
        continue;

      candidates.push(machine);
    }

    // Sort: prefer not-busy, then by free memory, then by load
    return candidates.sort((a, b) => {
      if (a.busy !== b.busy) return a.busy ? 1 : -1;
      const aLoad = a.resources.loadAvg?.[0] ?? 0;
      const bLoad = b.resources.loadAvg?.[0] ?? 0;
      return aLoad - bLoad;
    });
  }

  /**
   * Dispatch a request to the best available machine for a capability.
   */
  async dispatchToCapability(
    capabilityName: string,
    input: unknown,
    options?: { timeoutMs?: number; pipelineId?: string; stepIndex?: number }
  ): Promise<{ machineId: string; result: unknown }> {
    const candidates = this.findMachinesForCapability(capabilityName);

    if (candidates.length === 0) {
      throw new Error(
        `No machines available for capability "${capabilityName}"`
      );
    }

    const target = candidates[0];
    const ws = this.registry.connections.get(target.machineId);

    if (!ws) {
      throw new Error(
        `Machine "${target.machineId}" is registered but not connected`
      );
    }

    const requestId = crypto.randomUUID();

    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        reject(
          new Error(
            `Timeout waiting for response from "${target.machineId}" ` +
              `(capability: ${capabilityName})`
          )
        );
      }, options?.timeoutMs ?? 30_000);

      // Store the resolver so the WebSocket message handler can call it
      this.pendingRequests.set(requestId, {
        resolve: (result: unknown) => {
          clearTimeout(timeout);
          resolve({ machineId: target.machineId, result });
        },
        reject: (error: Error) => {
          clearTimeout(timeout);
          reject(error);
        },
      });

      const message: DispatchRequest = {
        type: "request",
        id: requestId,
        capability: capabilityName,
        input,
        pipelineId: options?.pipelineId,
        stepIndex: options?.stepIndex,
        timeoutMs: options?.timeoutMs,
      };

      ws.send(JSON.stringify(message));
    });
  }

  private pendingRequests = new Map<
    string,
    {
      resolve: (result: unknown) => void;
      reject: (error: Error) => void;
    }
  >();

  /**
   * Remove a machine from the registry and update the capability index.
   */
  private deregisterMachine(machineId: string): void {
    const machine = this.registry.machines.get(machineId);
    if (!machine) return;

    // Remove from capability index
    for (const cap of machine.capabilities) {
      const machines = this.registry.capabilityIndex.get(cap.name);
      if (machines) {
        machines.delete(machineId);
        if (machines.size === 0) {
          this.registry.capabilityIndex.delete(cap.name);
        }
      }
    }

    // Close WebSocket if still open
    const ws = this.registry.connections.get(machineId);
    if (ws) {
      try {
        ws.close(1000, "deregistered");
      } catch {
        // ignore
      }
      this.registry.connections.delete(machineId);
    }

    this.registry.machines.delete(machineId);
  }
}
```

### Connector-Side Handler

Here is how the connector processes incoming dispatch requests. This is a production-quality implementation based on the real Atlas connector:

```typescript
import * as os from "node:os";
import WebSocket from "ws";
import { z } from "zod";

// ── Capability Registry ─────────────────────────────────────────────────────

type CapabilityResult = {
  success: boolean;
  data?: unknown;
  error?: string;
};

type CapabilityHandler = (input: unknown) => Promise<CapabilityResult>;

const CAPABILITY_HANDLERS: Map<string, CapabilityHandler> = new Map();

/**
 * Register a capability handler. Called at startup.
 */
function registerCapability(
  name: string,
  handler: CapabilityHandler
): void {
  CAPABILITY_HANDLERS.set(name, handler);
}

// ── Built-in Capabilities ───────────────────────────────────────────────────

registerCapability("jane-status", async () => ({
  success: true,
  data: {
    hostname: os.hostname(),
    platform: os.platform(),
    arch: os.arch(),
    uptimeSeconds: os.uptime(),
    cpu: {
      model: os.cpus()[0]?.model ?? "unknown",
      cores: os.cpus().length,
      loadAvg: os.loadavg(),
    },
    memory: {
      totalGb: +(os.totalmem() / 1e9).toFixed(1),
      freeGb: +(os.freemem() / 1e9).toFixed(1),
      usedPct: +(
        ((os.totalmem() - os.freemem()) / os.totalmem()) *
        100
      ).toFixed(1),
    },
  },
}));

registerCapability("hud-notify", async (input) => {
  const schema = z.object({
    severity: z.enum(["green", "yellow", "red"]),
    message: z.string(),
    source: z.string().default("connector"),
  });

  const parsed = schema.safeParse(input);
  if (!parsed.success) {
    return { success: false, error: parsed.error.message };
  }

  // Write to the HUD status queue file
  const fs = await import("node:fs/promises");
  const path = await import("node:path");
  const queuePath = path.join(os.homedir(), ".atlas", "status-queue.json");

  const msg = {
    id: crypto.randomUUID(),
    ...parsed.data,
    created: new Date().toISOString(),
  };

  let queue = { messages: [] as unknown[] };
  try {
    const raw = await fs.readFile(queuePath, "utf-8");
    queue = JSON.parse(raw);
  } catch {
    // File doesn't exist — start fresh
  }

  queue.messages.push(msg);
  await fs.writeFile(queuePath, JSON.stringify(queue, null, 2));

  return { success: true, data: { id: msg.id } };
});

// ── Request Handler ─────────────────────────────────────────────────────────

async function handleDispatch(
  ws: WebSocket,
  request: DispatchRequest,
  machineId: string
): Promise<void> {
  const handler = CAPABILITY_HANDLERS.get(request.capability);

  if (!handler) {
    ws.send(
      JSON.stringify({
        type: "response",
        id: request.id,
        status: 404,
        body: {
          error: "unknown_capability",
          message: `Capability "${request.capability}" not registered`,
          available: Array.from(CAPABILITY_HANDLERS.keys()),
        },
      })
    );
    return;
  }

  // Report busy
  ws.send(
    JSON.stringify({
      type: "status-report",
      machine: machineId,
      busy: true,
      version: "0.1.0",
    })
  );

  let result: CapabilityResult;
  const startTime = Date.now();

  try {
    // Apply timeout if specified
    if (request.timeoutMs) {
      result = await Promise.race([
        handler(request.input),
        new Promise<CapabilityResult>((_, reject) =>
          setTimeout(
            () => reject(new Error("Capability execution timed out")),
            request.timeoutMs!
          )
        ),
      ]);
    } else {
      result = await handler(request.input);
    }
  } catch (err) {
    result = {
      success: false,
      error: `Handler threw: ${err instanceof Error ? err.message : String(err)}`,
    };
  }

  const durationMs = Date.now() - startTime;

  // Send response
  ws.send(
    JSON.stringify({
      type: "response",
      id: request.id,
      status: result.success ? 200 : 500,
      body: { ...result, durationMs },
    })
  );

  // Report not busy
  ws.send(
    JSON.stringify({
      type: "status-report",
      machine: machineId,
      busy: false,
      version: "0.1.0",
    })
  );
}
```

---

## Capability Discovery

### How Systems Discover What Machines Can Do

Capability discovery is the process of building and maintaining an index of "what can be done and where." Different systems take fundamentally different approaches.

### Pattern 1: Central Registry (Consul, etcd)

[Consul](https://developer.hashicorp.com/consul/docs/use-case/service-discovery) and [etcd](https://etcd.io/) use a central key-value store as the source of truth. Machines register themselves, and consumers query the registry.

```typescript
// Consul-style service registration
interface ConsulServiceRegistration {
  ID: string;
  Name: string;
  Tags: string[];
  Port: number;
  Meta: Record<string, string>;
  Check: {
    HTTP?: string;
    TCP?: string;
    Interval: string;
    Timeout: string;
    DeregisterCriticalServiceAfter: string;
  };
}

// Register a service with Consul
async function registerWithConsul(
  consulAddr: string,
  service: ConsulServiceRegistration
): Promise<void> {
  await fetch(`${consulAddr}/v1/agent/service/register`, {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(service),
  });
}

// Query capabilities from Consul
async function findCapableServices(
  consulAddr: string,
  capability: string
): Promise<ConsulServiceRegistration[]> {
  const res = await fetch(
    `${consulAddr}/v1/catalog/service/atlas-connector?tag=${capability}`
  );
  return res.json();
}
```

**Trade-offs:**
- Central point of truth — no split-brain
- Requires the registry to be available (SPoF without clustering)
- Health checks are poll-based (Consul) or TTL-based (etcd)
- Good for services that rarely change their capabilities

### Pattern 2: Gossip Protocol (Consul Serf, Swim)

Consul itself uses a [gossip protocol (Serf)](https://www.serf.io/) for cluster membership and failure detection. Nodes communicate directly with each other, spreading membership information epidemically.

```typescript
// Gossip-style peer discovery
interface GossipPeer {
  machineId: string;
  address: string;
  capabilities: string[];
  generation: number; // Lamport clock for conflict resolution
  status: "alive" | "suspect" | "dead";
  lastSeen: number;
}

class GossipMembership {
  private peers: Map<string, GossipPeer> = new Map();
  private self: GossipPeer;
  private fanout = 3; // Number of peers to gossip to per round

  constructor(selfPeer: GossipPeer) {
    this.self = selfPeer;
    this.peers.set(selfPeer.machineId, selfPeer);
  }

  /**
   * Gossip round: pick random peers and exchange membership lists.
   * Each round, every node talks to `fanout` other nodes.
   * Information propagates in O(log N) rounds.
   */
  async gossipRound(): Promise<void> {
    const alivePeers = Array.from(this.peers.values()).filter(
      (p) => p.status === "alive" && p.machineId !== this.self.machineId
    );

    // Pick random peers to gossip with
    const targets = this.pickRandom(alivePeers, this.fanout);

    for (const target of targets) {
      try {
        // Send our membership list, receive theirs
        const response = await this.sendGossip(target.address, {
          sender: this.self,
          members: Array.from(this.peers.values()),
        });

        // Merge received membership info
        for (const remotePeer of response.members) {
          this.mergePeer(remotePeer);
        }
      } catch {
        // Mark peer as suspect
        target.status = "suspect";
      }
    }
  }

  private mergePeer(remote: GossipPeer): void {
    const local = this.peers.get(remote.machineId);
    if (!local || remote.generation > local.generation) {
      this.peers.set(remote.machineId, remote);
    }
  }

  private pickRandom<T>(arr: T[], n: number): T[] {
    const shuffled = [...arr].sort(() => Math.random() - 0.5);
    return shuffled.slice(0, n);
  }

  private async sendGossip(
    address: string,
    payload: { sender: GossipPeer; members: GossipPeer[] }
  ): Promise<{ members: GossipPeer[] }> {
    const res = await fetch(`${address}/gossip`, {
      method: "POST",
      body: JSON.stringify(payload),
    });
    return res.json();
  }
}
```

### Pattern 3: mDNS / Bonjour (Zero-Configuration Discovery)

For local network discovery, [mDNS (multicast DNS)](https://datatracker.ietf.org/doc/html/rfc6762) and [DNS-SD (DNS Service Discovery)](https://datatracker.ietf.org/doc/html/rfc6763) let machines find each other without any central registry. On macOS, this is Bonjour. On Linux, it is Avahi.

```typescript
import { Bonjour } from "bonjour-service";

const bonjour = new Bonjour();

// Advertise this machine's capabilities on the local network
function advertiseCapabilities(
  machineId: string,
  capabilities: string[],
  port: number
): void {
  bonjour.publish({
    name: `atlas-connector-${machineId}`,
    type: "atlas-connector",
    protocol: "tcp",
    port,
    txt: {
      machineId,
      capabilities: capabilities.join(","),
      version: "0.1.0",
      role: "node",
    },
  });
}

// Discover other Atlas connectors on the local network
function discoverPeers(
  onFound: (peer: {
    machineId: string;
    host: string;
    port: number;
    capabilities: string[];
  }) => void
): void {
  bonjour.find({ type: "atlas-connector" }, (service) => {
    const txt = service.txt as Record<string, string>;
    onFound({
      machineId: txt.machineId ?? service.name,
      host: service.host,
      port: service.port,
      capabilities: (txt.capabilities ?? "").split(",").filter(Boolean),
    });
  });
}

// Usage
advertiseCapabilities("jane-macbook", ["hud-notify", "jane-status"], 7654);

discoverPeers((peer) => {
  console.log(
    `Found peer: ${peer.machineId} at ${peer.host}:${peer.port}`,
    `capabilities: ${peer.capabilities.join(", ")}`
  );
});
```

> **Key insight:** mDNS is perfect for LAN-only discovery but does not cross network boundaries. For a system like Atlas that spans local machines and cloud, you need both: mDNS for discovering peers on the same network, and a central registry (the hub) for cross-network coordination. Peers discovered via mDNS should still register with the hub.

### Pattern 4: Kubernetes Labels and Taints

[Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) uses node labels, taints, and tolerations for capability-based scheduling. Labels describe what a node _has_. Taints describe what a node _rejects_.

```typescript
// Kubernetes-inspired label-based capability matching
interface MachineLabels {
  // Standard labels
  "atlas.dev/platform": string;      // "darwin", "linux"
  "atlas.dev/arch": string;          // "arm64", "amd64"
  "atlas.dev/role": string;          // "node", "hub"

  // Capability labels
  "atlas.dev/gpu": string;           // "true", "false"
  "atlas.dev/gpu-model"?: string;    // "m2-ultra", "rtx-4090"
  "atlas.dev/gpu-vram"?: string;     // "64", "24" (GB)
  "atlas.dev/display": string;       // "true", "false"
  "atlas.dev/inference": string;     // "ollama", "vllm", "none"

  // Custom labels
  [key: string]: string | undefined;
}

// Taints: machine rejects work unless the request tolerates it
interface MachineTaint {
  key: string;
  value: string;
  effect: "NoSchedule" | "PreferNoSchedule" | "NoExecute";
}

// Selector: request specifies what it needs
interface CapabilitySelector {
  /** Required labels (AND logic) */
  matchLabels?: Record<string, string>;
  /** Label expressions (more flexible matching) */
  matchExpressions?: {
    key: string;
    operator: "In" | "NotIn" | "Exists" | "DoesNotExist" | "Gt" | "Lt";
    values?: string[];
  }[];
  /** Tolerate specific taints */
  tolerations?: {
    key: string;
    operator: "Equal" | "Exists";
    value?: string;
    effect?: MachineTaint["effect"];
  }[];
}

// Example: find a machine with GPU and display for inference + visualization
const inferenceSelector: CapabilitySelector = {
  matchLabels: {
    "atlas.dev/gpu": "true",
    "atlas.dev/display": "true",
  },
  matchExpressions: [
    {
      key: "atlas.dev/gpu-vram",
      operator: "Gt",
      values: ["16"], // At least 16GB VRAM
    },
    {
      key: "atlas.dev/inference",
      operator: "In",
      values: ["ollama", "vllm"],
    },
  ],
};

function matchesMachine(
  machine: { labels: MachineLabels; taints?: MachineTaint[] },
  selector: CapabilitySelector
): boolean {
  // Check matchLabels
  if (selector.matchLabels) {
    for (const [key, value] of Object.entries(selector.matchLabels)) {
      if ((machine.labels as Record<string, string>)[key] !== value) {
        return false;
      }
    }
  }

  // Check matchExpressions
  if (selector.matchExpressions) {
    for (const expr of selector.matchExpressions) {
      const machineValue = (machine.labels as Record<string, string>)[
        expr.key
      ];
      switch (expr.operator) {
        case "In":
          if (!expr.values?.includes(machineValue ?? "")) return false;
          break;
        case "NotIn":
          if (expr.values?.includes(machineValue ?? "")) return false;
          break;
        case "Exists":
          if (machineValue === undefined) return false;
          break;
        case "DoesNotExist":
          if (machineValue !== undefined) return false;
          break;
        case "Gt":
          if (Number(machineValue) <= Number(expr.values?.[0])) return false;
          break;
        case "Lt":
          if (Number(machineValue) >= Number(expr.values?.[0])) return false;
          break;
      }
    }
  }

  // Check taints vs tolerations
  if (machine.taints) {
    for (const taint of machine.taints) {
      if (taint.effect === "NoSchedule" || taint.effect === "NoExecute") {
        const tolerated = selector.tolerations?.some(
          (t) =>
            t.key === taint.key &&
            (t.operator === "Exists" || t.value === taint.value)
        );
        if (!tolerated) return false;
      }
    }
  }

  return true;
}
```

### Comparison: Discovery Approaches

| Approach | Scope | Latency | Dependencies | Best For |
|----------|-------|---------|-------------|----------|
| **Central Registry** (Consul/etcd) | Global | Low (single query) | Registry server must be available | Cloud-native services, cross-DC |
| **Gossip Protocol** (Serf/SWIM) | Cluster | O(log N) rounds | None (peer-to-peer) | Large clusters, failure detection |
| **mDNS/Bonjour** | LAN only | Sub-second | None (multicast) | Local network, zero-config |
| **K8s Labels** | Cluster | Low (API server query) | Kubernetes API server | Container workloads |
| **Tailscale** | Global mesh | Low | Tailscale coordination server | Secure cross-network connectivity |
| **Hub-spoke** (Atlas pattern) | Global | Low (hub query) | Hub must be available | AI agent platforms, mixed networks |

For Atlas, the answer is a hybrid: **mDNS for LAN peer discovery + cloud hub for global coordination**. Machines on the same network find each other instantly via mDNS. All machines register with the cloud hub for cross-network routing and pipeline orchestration.

---

## Node / Hub / Chain Architecture

### Node: The Worker

A node is a machine that registers capabilities and executes dispatched work. It does not make routing decisions — it receives requests and returns results.

```typescript
class AtlasNode {
  private config: ConnectorConfig;
  private ws: WebSocket | null = null;
  private capabilities: Map<string, CapabilityHandler> = new Map();
  private reconnectAttempt = 0;
  private heartbeatTimer: ReturnType<typeof setInterval> | null = null;

  constructor(config: ConnectorConfig) {
    this.config = config;
  }

  /**
   * Register a capability handler on this node.
   */
  addCapability(name: string, handler: CapabilityHandler): void {
    this.capabilities.set(name, handler);
  }

  /**
   * Start the node: register with hub, connect WebSocket, begin heartbeat.
   */
  async start(): Promise<void> {
    await this.register();
    this.connect();
  }

  private async register(): Promise<void> {
    const capabilities = Array.from(this.capabilities.keys()).map((name) => ({
      name,
      spec: { description: `Capability: ${name}` },
    }));

    const body = {
      machine_id: this.config.machineId,
      label: this.config.machineLabel,
      endpoint: this.config.runnerWsUrl,
      capabilities,
      resources: {
        platform: os.platform(),
        arch: os.arch(),
        cpus: os.cpus().length,
        memory_gb: +(os.totalmem() / 1e9).toFixed(1),
      },
      ttl_seconds: this.config.registrationTtlSeconds,
    };

    const res = await fetch(
      `${this.config.apiMomUrl}/v1/machines/register`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${this.config.secret}`,
        },
        body: JSON.stringify(body),
      }
    );

    if (!res.ok) {
      throw new Error(`Registration failed: ${res.status}`);
    }
  }

  private connect(): void {
    this.ws = new WebSocket(this.config.runnerWsUrl, {
      headers: { Authorization: `Bearer ${this.config.secret}` },
    });

    this.ws.on("open", () => {
      this.reconnectAttempt = 0;
      this.startHeartbeat();
    });

    this.ws.on("message", (raw) => {
      const msg = JSON.parse(raw.toString());
      if (msg.type === "request") {
        this.handleRequest(msg);
      } else if (msg.type === "ping") {
        this.ws?.send(JSON.stringify({ type: "pong" }));
      }
    });

    this.ws.on("close", () => {
      this.stopHeartbeat();
      this.scheduleReconnect();
    });
  }

  private async handleRequest(msg: DispatchRequest): Promise<void> {
    const handler = this.capabilities.get(msg.capability);
    if (!handler) {
      this.ws?.send(
        JSON.stringify({
          type: "response",
          id: msg.id,
          status: 404,
          body: { error: "unknown_capability" },
        })
      );
      return;
    }

    const result = await handler(msg.input);
    this.ws?.send(
      JSON.stringify({
        type: "response",
        id: msg.id,
        status: result.success ? 200 : 500,
        body: result,
      })
    );
  }

  private startHeartbeat(): void {
    this.heartbeatTimer = setInterval(async () => {
      await fetch(`${this.config.apiMomUrl}/v1/machines/heartbeat`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${this.config.secret}`,
        },
        body: JSON.stringify({
          machine_id: this.config.machineId,
          capabilities: Array.from(this.capabilities.keys()),
          resources: {
            platform: os.platform(),
            cpus: os.cpus().length,
            memory_gb: +(os.totalmem() / 1e9).toFixed(1),
            free_memory_gb: +(os.freemem() / 1e9).toFixed(1),
            load_avg: os.loadavg(),
          },
        }),
      }).catch(() => {});
    }, this.config.heartbeatIntervalMs);
  }

  private stopHeartbeat(): void {
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
  }

  private scheduleReconnect(): void {
    const delay = Math.min(1000 * 2 ** this.reconnectAttempt, 60_000);
    const jitter = delay * (0.8 + Math.random() * 0.4);
    this.reconnectAttempt++;
    setTimeout(() => this.connect(), jitter);
  }
}
```

### Hub: The Coordinator

The hub maintains the machine registry, routes capability requests, and orchestrates pipelines. In Atlas, the hub is a Cloudflare Durable Object — it is always available, globally distributed, and maintains persistent state.

```typescript
class AtlasHub {
  private machines: Map<string, RegisteredMachine> = new Map();
  private capabilityIndex: Map<string, Set<string>> = new Map();
  private connections: Map<string, WebSocket> = new Map();
  private pipelines: Map<string, PipelineDefinition> = new Map();

  /**
   * Register a machine. Called via POST /v1/machines/register.
   */
  registerMachine(registration: MachineRegistration): void {
    const machine: RegisteredMachine = {
      ...registration,
      registeredAt: Date.now(),
      lastHeartbeat: Date.now(),
      busy: false,
      activeTasks: [],
      version: "0.1.0",
      connected: false,
    };

    this.machines.set(registration.machineId, machine);

    // Update capability index
    for (const cap of registration.capabilities) {
      if (!this.capabilityIndex.has(cap.name)) {
        this.capabilityIndex.set(cap.name, new Set());
      }
      this.capabilityIndex.get(cap.name)!.add(registration.machineId);
    }
  }

  /**
   * Get the full capability map — what's available across all machines.
   */
  getCapabilityMap(): Map<string, string[]> {
    const map = new Map<string, string[]>();
    for (const [capability, machineIds] of this.capabilityIndex) {
      map.set(capability, Array.from(machineIds));
    }
    return map;
  }

  /**
   * Execute a pipeline by dispatching each step to the appropriate machine.
   */
  async executePipeline(
    pipelineId: string,
    initialInput: unknown
  ): Promise<PipelineResult> {
    const pipeline = this.pipelines.get(pipelineId);
    if (!pipeline) {
      throw new Error(`Pipeline "${pipelineId}" not found`);
    }

    let currentInput = initialInput;
    const stepResults: StepResult[] = [];

    for (let i = 0; i < pipeline.steps.length; i++) {
      const step = pipeline.steps[i];

      try {
        const { machineId, result } = await this.dispatchToCapability(
          step.capability,
          currentInput,
          {
            timeoutMs: step.timeoutMs,
            pipelineId,
            stepIndex: i,
          }
        );

        const stepResult: StepResult = {
          step: i,
          capability: step.capability,
          machineId,
          status: "completed",
          result,
          durationMs: 0, // Would be filled from timing
        };

        stepResults.push(stepResult);

        // Transform output for next step
        if (step.outputTransform) {
          currentInput = step.outputTransform(result);
        } else {
          currentInput = result;
        }
      } catch (error) {
        const stepResult: StepResult = {
          step: i,
          capability: step.capability,
          machineId: "none",
          status: "failed",
          error: error instanceof Error ? error.message : String(error),
          durationMs: 0,
        };

        stepResults.push(stepResult);

        // Check retry policy
        if (step.retryPolicy && step.retryPolicy.maxRetries > 0) {
          // Retry logic would go here
        }

        if (!step.continueOnFailure) {
          return {
            pipelineId,
            status: "failed",
            steps: stepResults,
            failedAt: i,
          };
        }
      }
    }

    return {
      pipelineId,
      status: "completed",
      steps: stepResults,
      output: currentInput,
    };
  }

  // ... dispatchToCapability from earlier
  async dispatchToCapability(
    capabilityName: string,
    input: unknown,
    options?: { timeoutMs?: number; pipelineId?: string; stepIndex?: number }
  ): Promise<{ machineId: string; result: unknown }> {
    // Implementation from the Registry section above
    throw new Error("See full implementation in Registry section");
  }
}
```

### Chain: The Pipeline

A chain is a sequence of capabilities that are executed in order, with the output of each step becoming the input of the next. The chain definition is declarative — it says _what_ to do, not _where_ to do it. The hub resolves each step to a machine at execution time.

```typescript
interface PipelineDefinition {
  id: string;
  name: string;
  description: string;
  steps: PipelineStep[];
  /** Global timeout for the entire pipeline */
  timeoutMs?: number;
  /** Retry the entire pipeline on failure */
  retryPolicy?: RetryPolicy;
}

interface PipelineStep {
  /** Capability to invoke */
  capability: string;
  /** Optional: override the input (instead of using previous step's output) */
  inputOverride?: unknown;
  /** Transform the output before passing to the next step */
  outputTransform?: (output: unknown) => unknown;
  /** Timeout for this step */
  timeoutMs?: number;
  /** Retry policy for this step */
  retryPolicy?: RetryPolicy;
  /** Continue the pipeline even if this step fails */
  continueOnFailure?: boolean;
  /** Capability selector constraints (e.g., "must have GPU") */
  selector?: CapabilitySelector;
}

interface RetryPolicy {
  maxRetries: number;
  backoffMs: number;
  backoffMultiplier: number;
  maxBackoffMs: number;
}

interface StepResult {
  step: number;
  capability: string;
  machineId: string;
  status: "completed" | "failed" | "skipped";
  result?: unknown;
  error?: string;
  durationMs: number;
}

interface PipelineResult {
  pipelineId: string;
  status: "completed" | "failed";
  steps: StepResult[];
  output?: unknown;
  failedAt?: number;
}

// ── Example Pipeline Definitions ────────────────────────────────────────────

// Pipeline: Scrape → Summarize → Notify
const scrapeAndNotifyPipeline: PipelineDefinition = {
  id: "scrape-summarize-notify",
  name: "Scrape and Notify",
  description:
    "Scrape a URL, summarize the content, and send a HUD notification",
  steps: [
    {
      capability: "web-scrape",
      timeoutMs: 30_000,
      retryPolicy: {
        maxRetries: 2,
        backoffMs: 1000,
        backoffMultiplier: 2,
        maxBackoffMs: 10_000,
      },
    },
    {
      capability: "llm-summarize",
      selector: {
        matchLabels: { "atlas.dev/inference": "ollama" },
      },
      timeoutMs: 60_000,
    },
    {
      capability: "hud-notify",
      outputTransform: (summary) => ({
        severity: "green",
        message: `Summary: ${(summary as { text: string }).text}`,
        source: "pipeline/scrape-summarize-notify",
      }),
      selector: {
        matchLabels: { "atlas.dev/display": "true" },
      },
    },
  ],
  timeoutMs: 120_000,
};

// Pipeline: Inference → Validate → Deploy
const inferenceDeployPipeline: PipelineDefinition = {
  id: "inference-validate-deploy",
  name: "Inference and Deploy",
  description: "Run inference, validate the result, deploy to production",
  steps: [
    {
      capability: "gpu-inference",
      selector: {
        matchLabels: { "atlas.dev/gpu": "true" },
        matchExpressions: [
          { key: "atlas.dev/gpu-vram", operator: "Gt", values: ["8"] },
        ],
      },
      timeoutMs: 120_000,
    },
    {
      capability: "validate-output",
      timeoutMs: 10_000,
    },
    {
      capability: "deploy-result",
      timeoutMs: 30_000,
      retryPolicy: {
        maxRetries: 3,
        backoffMs: 2000,
        backoffMultiplier: 2,
        maxBackoffMs: 30_000,
      },
    },
  ],
};
```

### How Temporal Does It

[Temporal.io](https://temporal.io/) uses a similar pattern with [Task Queues](https://docs.temporal.io/task-queue). Workers subscribe to task queues. Workflows dispatch activities to queues. The Temporal server routes tasks to available workers.

The key differences from the Atlas approach:

| Aspect | Temporal | Atlas |
|--------|----------|-------|
| **Worker binding** | Workers poll specific task queues | Machines register capabilities, hub pushes work |
| **Task routing** | Queue name is the routing key | Capability name + label selector |
| **State persistence** | Temporal server (Cassandra/PostgreSQL) | Durable Object (Cloudflare) |
| **Failure handling** | Built-in activity retries, heartbeats, timeouts | Custom retry policy per step |
| **Workflow definition** | Code (TypeScript/Go/Java SDK) | Declarative pipeline JSON |

Temporal's approach is more mature and battle-tested for complex workflows. Atlas's approach is simpler, cloud-native (Cloudflare DOs), and designed specifically for AI agent coordination rather than general-purpose workflow orchestration.

### How Airflow Does It

[Apache Airflow](https://airflow.apache.org/) uses DAGs (Directed Acyclic Graphs) to define pipelines. Each task in the DAG is assigned to an executor pool. The scheduler determines when tasks run based on dependencies and pool availability.

```python
# Airflow DAG equivalent of the Atlas pipeline above
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG("scrape_summarize_notify", start_date=datetime(2026, 1, 1)) as dag:
    scrape = PythonOperator(
        task_id="web_scrape",
        python_callable=scrape_url,
        pool="network_workers",  # Airflow's version of capability routing
    )
    summarize = PythonOperator(
        task_id="llm_summarize",
        python_callable=summarize_text,
        pool="gpu_workers",
    )
    notify = PythonOperator(
        task_id="hud_notify",
        python_callable=send_notification,
        pool="display_workers",
    )
    scrape >> summarize >> notify
```

Airflow pools are the closest analog to Atlas capability routing, but they are static (configured in the Airflow metadata database) while Atlas capabilities are dynamic (discovered at registration time).

---

## Security

### The Machine Identity Problem

When a new machine joins the network, three things must be true:

1. **The machine is who it claims to be.** A malicious machine cannot impersonate a legitimate one.
2. **The hub is who it claims to be.** A MITM cannot intercept the registration and redirect the machine to a fake hub.
3. **Communication is encrypted.** No eavesdropping on capability invocations.

### Token-Based Authentication (Current Atlas Pattern)

The simplest approach — and the one used by [k3s](https://docs.k3s.io/), Atlas, and many distributed systems — is pre-shared tokens.

```typescript
// The install script receives a token
// curl -sfL https://install.atlas.dev | sh -s -- --token "$ATLAS_TOKEN"

// The connector uses it for all communication
const config: ConnectorConfig = {
  apiMomUrl: "https://api-mom.garywu.dev",
  secret: process.env.ATLAS_TOKEN!, // Pre-shared secret
  machineId: readMachineId(),
  // ...
};

// Registration: Bearer token in Authorization header
await fetch(`${config.apiMomUrl}/v1/machines/register`, {
  headers: {
    Authorization: `Bearer ${config.secret}`,
  },
  body: JSON.stringify(registrationPayload),
});

// WebSocket: Bearer token in handshake
const ws = new WebSocket(config.runnerWsUrl, {
  headers: {
    Authorization: `Bearer ${config.secret}`,
  },
});
```

**Limitations:**
- Single shared secret — if leaked, all machines are compromised
- No per-machine identity — any machine with the token can impersonate any other
- No rotation mechanism — changing the token requires updating all machines

### Per-Machine Keys with Registration Tokens

A better approach: use a one-time registration token to bootstrap per-machine credentials.

```typescript
interface RegistrationFlow {
  // Step 1: Admin generates a one-time registration token
  // POST /v1/admin/registration-tokens
  // Returns: { token: "reg_abc123...", expiresAt: "...", maxUses: 1 }

  // Step 2: Machine uses the token to register and get permanent credentials
  // POST /v1/machines/register
  // Authorization: Bearer reg_abc123...
  // Returns: { machineToken: "mach_xyz...", machineId: "...", certPem: "..." }

  // Step 3: Machine uses its permanent token for all further communication
  // Authorization: Bearer mach_xyz...
}

// Implementation
async function bootstrapMachineCredentials(
  hubUrl: string,
  registrationToken: string,
  machineId: string,
  publicKey: string
): Promise<{
  machineToken: string;
  hubPublicKey: string;
  certificate: string;
}> {
  const res = await fetch(`${hubUrl}/v1/machines/register`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${registrationToken}`,
    },
    body: JSON.stringify({
      machine_id: machineId,
      public_key: publicKey,
      // Challenge: prove we have the private key
      challenge_response: await signChallenge(machineId),
    }),
  });

  if (res.status === 401) {
    throw new Error("Registration token expired or invalid");
  }

  const credentials = (await res.json()) as {
    machine_token: string;
    hub_public_key: string;
    certificate: string;
  };

  // Store credentials securely
  const fs = await import("node:fs/promises");
  const path = await import("node:path");
  const credPath = path.join(os.homedir(), ".atlas", "credentials.json");

  await fs.writeFile(
    credPath,
    JSON.stringify(
      {
        machineId,
        machineToken: credentials.machine_token,
        hubPublicKey: credentials.hub_public_key,
        certificate: credentials.certificate,
        issuedAt: new Date().toISOString(),
      },
      null,
      2
    ),
    { mode: 0o600 }
  );

  return {
    machineToken: credentials.machine_token,
    hubPublicKey: credentials.hub_public_key,
    certificate: credentials.certificate,
  };
}
```

### mTLS for Machine-to-Machine Communication

For the highest security, machines use mutual TLS (mTLS) to authenticate to each other. The hub acts as a Certificate Authority (CA), issuing certificates during registration.

```typescript
import * as tls from "node:tls";
import * as fs from "node:fs";

// Machine-to-machine connection with mTLS
function createMtlsConnection(
  targetHost: string,
  targetPort: number
): tls.TLSSocket {
  const atlasDir = `${os.homedir()}/.atlas`;

  return tls.connect({
    host: targetHost,
    port: targetPort,
    // Our certificate (issued by the hub CA)
    cert: fs.readFileSync(`${atlasDir}/certs/machine.crt`),
    key: fs.readFileSync(`${atlasDir}/keys/machine.key`),
    // Trust only the hub's CA
    ca: fs.readFileSync(`${atlasDir}/certs/hub-ca.crt`),
    // Reject connections to machines not signed by the hub CA
    rejectUnauthorized: true,
    // Verify the peer's certificate
    requestCert: true,
  });
}

// Hub-side: issue a machine certificate during registration
async function issueMachineCertificate(
  machineId: string,
  publicKey: string,
  capabilities: string[]
): Promise<string> {
  // In practice, you would use a proper CA library (e.g., step-ca, CFSSL)
  // This is the concept:
  const certificate = await signCertificate({
    subject: {
      CN: machineId,
      O: "Atlas Network",
      OU: capabilities.join(","),
    },
    publicKey,
    validity: {
      notBefore: new Date(),
      notAfter: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000), // 1 year
    },
    extensions: {
      // Machine capabilities encoded in the certificate
      subjectAltName: capabilities.map((c) => `URI:atlas:capability:${c}`),
    },
  });

  return certificate;
}
```

### Token Rotation

Tokens should rotate periodically. The heartbeat is the natural rotation point.

```typescript
interface TokenRotation {
  /** Current token */
  currentToken: string;
  /** Previous token (still accepted for a grace period) */
  previousToken?: string;
  /** When the current token was issued */
  issuedAt: number;
  /** When the current token expires */
  expiresAt: number;
  /** Grace period: how long previousToken is still accepted (ms) */
  gracePeriodMs: number;
}

// Hub-side: rotate tokens during heartbeat
function handleHeartbeat(
  machineId: string,
  token: string,
  rotation: TokenRotation
): HeartbeatResponse {
  const now = Date.now();

  // Check if token needs rotation
  const tokenAge = now - rotation.issuedAt;
  const maxTokenAge = 24 * 60 * 60 * 1000; // 24 hours

  if (tokenAge > maxTokenAge) {
    const newToken = generateSecureToken();

    return {
      registered: true,
      tokenRotation: {
        newToken,
        effectiveAt: new Date(now + 60_000).toISOString(), // 1 min grace
        oldTokenExpiresAt: new Date(now + 5 * 60_000).toISOString(), // 5 min
      },
    };
  }

  return { registered: true };
}

// Connector-side: handle token rotation from heartbeat response
async function applyTokenRotation(
  response: HeartbeatResponse,
  configPath: string
): Promise<void> {
  if (!response.tokenRotation) return;

  const { newToken, effectiveAt } = response.tokenRotation;

  // Schedule the token switch
  const switchAt = new Date(effectiveAt).getTime() - Date.now();

  setTimeout(async () => {
    const config = JSON.parse(
      await fs.readFile(configPath, "utf-8")
    );
    config.previousToken = config.token;
    config.token = newToken;
    await fs.writeFile(configPath, JSON.stringify(config, null, 2));

    console.log("[connector] Token rotated successfully");
  }, Math.max(0, switchAt));
}
```

### Security Comparison

| Approach | Complexity | Security | Operations Cost |
|----------|-----------|----------|----------------|
| **Shared secret** (current Atlas) | Low | Low — one leaked token compromises everything | Low — one env var |
| **Per-machine tokens** (k3s pattern) | Medium | Medium — leaked token compromises one machine | Medium — manage per-machine tokens |
| **Registration token + per-machine creds** | Medium-High | High — registration tokens are ephemeral, machine tokens are unique | Medium — token rotation needed |
| **mTLS with hub CA** | High | Very High — certificates, mutual auth, capability encoding | High — CA management, cert renewal |
| **Tailscale** (WireGuard mesh) | Low (outsourced) | Very High — WireGuard encryption, Tailscale identity | Low — Tailscale handles everything |

> **Key insight:** Start with per-machine tokens issued via one-time registration codes. This gives you 80% of the security of mTLS with 20% of the operational complexity. If you need cross-machine direct communication (not just hub-to-node), add Tailscale as the transport layer rather than building your own mTLS infrastructure.

---

## Patterns

### Pattern 1: Exponential Backoff with Jitter

When a connector loses its WebSocket connection, it must reconnect — but not all at once. If the hub goes down and comes back, every machine reconnecting simultaneously creates a "thundering herd" that can crash the hub again.

```typescript
class ReconnectStrategy {
  private attempt = 0;
  private readonly baseDelayMs: number;
  private readonly maxDelayMs: number;
  private readonly jitterFactor: number;

  constructor(options?: {
    baseDelayMs?: number;
    maxDelayMs?: number;
    jitterFactor?: number;
  }) {
    this.baseDelayMs = options?.baseDelayMs ?? 1000;
    this.maxDelayMs = options?.maxDelayMs ?? 60_000;
    this.jitterFactor = options?.jitterFactor ?? 0.4; // +/- 20%
  }

  /**
   * Get the next delay, then increment the attempt counter.
   * Exponential backoff: delay = base * 2^attempt
   * With jitter: delay * (1 - jitterFactor/2 + random * jitterFactor)
   */
  nextDelay(): number {
    const exponential = this.baseDelayMs * 2 ** this.attempt;
    const capped = Math.min(exponential, this.maxDelayMs);
    const jitterMin = 1 - this.jitterFactor / 2;
    const jitter = jitterMin + Math.random() * this.jitterFactor;
    const delay = capped * jitter;

    this.attempt++;
    return Math.round(delay);
  }

  reset(): void {
    this.attempt = 0;
  }

  get currentAttempt(): number {
    return this.attempt;
  }
}

// Usage in the connector
const reconnect = new ReconnectStrategy({
  baseDelayMs: 1000,
  maxDelayMs: 60_000,
  jitterFactor: 0.4,
});

function scheduleReconnect(): void {
  const delay = reconnect.nextDelay();
  console.log(
    `[connector] reconnecting in ${(delay / 1000).toFixed(1)}s ` +
      `(attempt ${reconnect.currentAttempt})`
  );
  setTimeout(() => connect(), delay);
}

// On successful connection:
ws.on("open", () => {
  reconnect.reset();
  // ...
});
```

**When to use:** Every WebSocket, HTTP long-poll, or TCP connection that needs automatic reconnection.

**Gotcha:** Without jitter, all machines that disconnected at the same time will reconnect at the same time. The jitter spreads the reconnection window, preventing thundering herd.

### Pattern 2: Capability-Gated Dispatch

Route work to machines based on declared capabilities, not hardcoded addresses.

```typescript
class CapabilityRouter {
  private registry: MachineRegistry;

  constructor(registry: MachineRegistry) {
    this.registry = registry;
  }

  /**
   * Find the best machine for a capability.
   * Selection criteria (in priority order):
   * 1. Machine has the capability
   * 2. Machine is not busy
   * 3. Machine meets resource requirements
   * 4. Machine has the lowest load average
   */
  selectMachine(
    capability: string,
    requirements?: {
      gpu?: boolean;
      minMemoryGb?: number;
      preferLocal?: boolean;
    }
  ): RegisteredMachine | null {
    const candidates = this.findMachinesForCapability(capability);

    if (candidates.length === 0) return null;

    // Filter by requirements
    let filtered = candidates;
    if (requirements?.gpu) {
      filtered = filtered.filter(
        (m) => (m.resources.gpus?.length ?? 0) > 0
      );
    }
    if (requirements?.minMemoryGb) {
      filtered = filtered.filter(
        (m) => m.resources.freeMemoryGb >= requirements.minMemoryGb!
      );
    }

    if (filtered.length === 0) return null;

    // Score each candidate
    const scored = filtered.map((machine) => {
      let score = 100;

      // Penalty for being busy
      if (machine.busy) score -= 50;

      // Penalty for high load
      const load = machine.resources.loadAvg?.[0] ?? 0;
      const loadPerCore = load / machine.resources.cpus;
      score -= loadPerCore * 20;

      // Penalty for low memory
      const memPct =
        machine.resources.freeMemoryGb / machine.resources.memoryGb;
      if (memPct < 0.2) score -= 30;

      // Bonus for local machine (if preferLocal)
      if (requirements?.preferLocal) {
        // Machine is on the same network (detected via mDNS)
        // This is a heuristic — real implementation would check network
        score += 10;
      }

      return { machine, score };
    });

    // Sort by score descending
    scored.sort((a, b) => b.score - a.score);

    return scored[0]?.machine ?? null;
  }

  private findMachinesForCapability(
    capability: string
  ): RegisteredMachine[] {
    const machineIds = this.registry.capabilityIndex.get(capability);
    if (!machineIds) return [];

    const now = Date.now();
    return Array.from(machineIds)
      .map((id) => this.registry.machines.get(id))
      .filter((m): m is RegisteredMachine => {
        if (!m) return false;
        if (!m.connected) return false;
        const expiresAt = m.lastHeartbeat + m.ttlSeconds * 1000;
        return now < expiresAt;
      });
  }
}
```

**When to use:** Any time you need to dispatch work to heterogeneous machines. The caller says _what_ they need, not _where_ to send it.

**Gotcha:** Capability routing is only as good as the capability declarations. If a machine declares "gpu-inference" but its VRAM is full, the dispatch will fail. Capabilities must be dynamically validated.

### Pattern 3: Pipeline Checkpoint and Resume

Long-running pipelines need checkpointing so that if a step fails partway through, the pipeline can resume from the last successful step rather than starting over.

```typescript
interface PipelineCheckpoint {
  pipelineId: string;
  executionId: string;
  currentStep: number;
  stepResults: StepResult[];
  /** Input for the next step (output of the last successful step) */
  nextInput: unknown;
  startedAt: string;
  lastCheckpointAt: string;
  status: "running" | "paused" | "failed" | "completed";
}

class CheckpointedPipeline {
  private storage: DurableObjectStorage; // Or any KV store

  constructor(storage: DurableObjectStorage) {
    this.storage = storage;
  }

  /**
   * Execute a pipeline with checkpointing.
   * If a previous execution exists and was interrupted, resume from the
   * last checkpoint.
   */
  async execute(
    pipeline: PipelineDefinition,
    initialInput: unknown,
    executionId?: string
  ): Promise<PipelineResult> {
    const execId = executionId ?? crypto.randomUUID();

    // Check for existing checkpoint
    let checkpoint = await this.loadCheckpoint(execId);

    if (checkpoint && checkpoint.status === "running") {
      console.log(
        `[pipeline] Resuming execution ${execId} from step ${checkpoint.currentStep}`
      );
    } else {
      checkpoint = {
        pipelineId: pipeline.id,
        executionId: execId,
        currentStep: 0,
        stepResults: [],
        nextInput: initialInput,
        startedAt: new Date().toISOString(),
        lastCheckpointAt: new Date().toISOString(),
        status: "running",
      };
    }

    // Execute remaining steps
    for (
      let i = checkpoint.currentStep;
      i < pipeline.steps.length;
      i++
    ) {
      const step = pipeline.steps[i];

      try {
        const result = await this.executeStep(
          step,
          checkpoint.nextInput,
          pipeline.id,
          i
        );

        checkpoint.stepResults.push(result);
        checkpoint.nextInput = result.result;
        checkpoint.currentStep = i + 1;
        checkpoint.lastCheckpointAt = new Date().toISOString();

        // Persist checkpoint after each step
        await this.saveCheckpoint(checkpoint);
      } catch (error) {
        checkpoint.status = "failed";
        await this.saveCheckpoint(checkpoint);

        return {
          pipelineId: pipeline.id,
          status: "failed",
          steps: checkpoint.stepResults,
          failedAt: i,
        };
      }
    }

    checkpoint.status = "completed";
    await this.saveCheckpoint(checkpoint);

    return {
      pipelineId: pipeline.id,
      status: "completed",
      steps: checkpoint.stepResults,
      output: checkpoint.nextInput,
    };
  }

  private async executeStep(
    step: PipelineStep,
    input: unknown,
    pipelineId: string,
    stepIndex: number
  ): Promise<StepResult> {
    const startTime = Date.now();

    // Retry loop
    let lastError: Error | null = null;
    const maxRetries = step.retryPolicy?.maxRetries ?? 0;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        const { machineId, result } = await this.dispatch(
          step.capability,
          input,
          { timeoutMs: step.timeoutMs, pipelineId, stepIndex }
        );

        return {
          step: stepIndex,
          capability: step.capability,
          machineId,
          status: "completed",
          result: step.outputTransform
            ? step.outputTransform(result)
            : result,
          durationMs: Date.now() - startTime,
        };
      } catch (err) {
        lastError = err instanceof Error ? err : new Error(String(err));

        if (attempt < maxRetries) {
          const backoff = Math.min(
            (step.retryPolicy?.backoffMs ?? 1000) *
              (step.retryPolicy?.backoffMultiplier ?? 2) ** attempt,
            step.retryPolicy?.maxBackoffMs ?? 30_000
          );
          await new Promise((resolve) => setTimeout(resolve, backoff));
        }
      }
    }

    throw lastError;
  }

  private async dispatch(
    capability: string,
    input: unknown,
    options: { timeoutMs?: number; pipelineId: string; stepIndex: number }
  ): Promise<{ machineId: string; result: unknown }> {
    // Delegate to the hub's dispatch mechanism
    throw new Error("Implement via hub reference");
  }

  private async loadCheckpoint(
    executionId: string
  ): Promise<PipelineCheckpoint | null> {
    return (
      (await this.storage.get<PipelineCheckpoint>(
        `checkpoint:${executionId}`
      )) ?? null
    );
  }

  private async saveCheckpoint(
    checkpoint: PipelineCheckpoint
  ): Promise<void> {
    await this.storage.put(
      `checkpoint:${checkpoint.executionId}`,
      checkpoint
    );
  }
}
```

**When to use:** Any multi-step pipeline where individual steps are expensive (GPU inference, large data transfers) and re-execution is wasteful.

**Connection to other patterns:** Checkpointing uses the same storage layer as the hub's machine registry. Both are Durable Object state. The pipeline executor references the capability router (Pattern 2) for dispatch.

### Pattern 4: Fan-Out / Fan-In

When a step needs to run on multiple machines in parallel (e.g., distributed inference across GPUs), use fan-out to dispatch to all capable machines and fan-in to collect results.

```typescript
interface FanOutStep {
  capability: string;
  /** How to split the input for parallel execution */
  splitter: (input: unknown) => unknown[];
  /** How to combine the results */
  reducer: (results: unknown[]) => unknown;
  /** Maximum parallelism */
  maxParallel?: number;
  /** Minimum machines required (fail if fewer available) */
  minMachines?: number;
}

async function executeFanOut(
  step: FanOutStep,
  input: unknown,
  router: CapabilityRouter,
  registry: MachineRegistry
): Promise<{ results: unknown[]; combined: unknown }> {
  // Split input into chunks
  const chunks = step.splitter(input);

  // Find all machines with this capability
  const machines = router.findAllMachinesForCapability(step.capability);

  if (step.minMachines && machines.length < step.minMachines) {
    throw new Error(
      `Fan-out requires ${step.minMachines} machines for "${step.capability}", ` +
        `but only ${machines.length} available`
    );
  }

  // Limit parallelism
  const maxParallel = step.maxParallel ?? machines.length;
  const concurrency = Math.min(maxParallel, machines.length, chunks.length);

  // Distribute chunks across machines (round-robin)
  const assignments: { machine: RegisteredMachine; chunk: unknown }[] =
    chunks.map((chunk, i) => ({
      machine: machines[i % machines.length],
      chunk,
    }));

  // Execute in parallel with concurrency limit
  const results: unknown[] = [];
  const executing = new Set<Promise<void>>();

  for (const assignment of assignments) {
    const promise = (async () => {
      const ws = registry.connections.get(assignment.machine.machineId);
      if (!ws) throw new Error("Machine disconnected during fan-out");

      const requestId = crypto.randomUUID();
      const result = await dispatchAndWait(ws, {
        type: "request",
        id: requestId,
        capability: step.capability,
        input: assignment.chunk,
      });

      results.push(result);
    })();

    executing.add(promise);
    promise.finally(() => executing.delete(promise));

    if (executing.size >= concurrency) {
      await Promise.race(executing);
    }
  }

  // Wait for all remaining
  await Promise.all(executing);

  // Reduce results
  const combined = step.reducer(results);

  return { results, combined };
}

// Example: Distributed text embedding across GPU machines
const embeddingFanOut: FanOutStep = {
  capability: "text-embedding",
  splitter: (input) => {
    const texts = (input as { texts: string[] }).texts;
    // Split into chunks of 100
    const chunks: unknown[] = [];
    for (let i = 0; i < texts.length; i += 100) {
      chunks.push({ texts: texts.slice(i, i + 100) });
    }
    return chunks;
  },
  reducer: (results) => ({
    embeddings: results.flatMap(
      (r) => (r as { embeddings: number[][] }).embeddings
    ),
  }),
  maxParallel: 4,
  minMachines: 1,
};
```

### Pattern 5: Self-Update via Hub Command

The hub can instruct machines to upgrade themselves. This uses the heartbeat response as a command channel.

```typescript
// Hub side: schedule an update rollout
class UpdateRollout {
  private machines: Map<string, RegisteredMachine>;
  private targetVersion: string;
  private downloadUrl: string;
  private rollingPercentage: number;
  private updatedMachines: Set<string> = new Set();

  constructor(
    machines: Map<string, RegisteredMachine>,
    targetVersion: string,
    downloadUrl: string,
    rollingPercentage = 10 // Update 10% at a time
  ) {
    this.machines = machines;
    this.targetVersion = targetVersion;
    this.downloadUrl = downloadUrl;
    this.rollingPercentage = rollingPercentage;
  }

  /**
   * Called on each heartbeat. Returns update command if this machine
   * should update.
   */
  shouldUpdate(machineId: string, currentVersion: string): UpdateCommand | null {
    if (currentVersion === this.targetVersion) {
      this.updatedMachines.add(machineId);
      return null;
    }

    // Check if we have capacity to update more machines
    const totalMachines = this.machines.size;
    const currentlyUpdating = Array.from(this.machines.values()).filter(
      (m) => m.version !== this.targetVersion && !this.updatedMachines.has(m.machineId)
    ).length;

    const maxUpdating = Math.ceil(
      (totalMachines * this.rollingPercentage) / 100
    );

    if (currentlyUpdating >= maxUpdating) {
      return null; // Wait for current batch to finish
    }

    return {
      type: "update",
      version: this.targetVersion,
      url: this.downloadUrl,
      restart: true,
    };
  }
}

// Connector side: handle update command
async function handleUpdate(command: UpdateCommand): Promise<void> {
  if (!command.version || !command.url) {
    console.warn("[connector] update command missing version or URL");
    return;
  }

  console.log(`[connector] Updating to version ${command.version}...`);

  const tmpDir = await fs.mkdtemp(path.join(os.tmpdir(), "atlas-update-"));

  try {
    // Download new binary
    const response = await fetch(command.url);
    const buffer = Buffer.from(await response.arrayBuffer());
    const tmpPath = path.join(tmpDir, "atlas-connector");
    await fs.writeFile(tmpPath, buffer);
    await fs.chmod(tmpPath, 0o755);

    // Verify the new binary runs
    const { execFile } = await import("node:child_process");
    await new Promise<void>((resolve, reject) => {
      execFile(tmpPath, ["--version"], { timeout: 5000 }, (err, stdout) => {
        if (err) reject(err);
        else {
          console.log(`[connector] New binary version: ${stdout.trim()}`);
          resolve();
        }
      });
    });

    // Replace the current binary
    const currentPath = process.argv[1]; // Or configured install path
    await fs.copyFile(tmpPath, currentPath);
    await fs.chmod(currentPath, 0o755);

    console.log("[connector] Binary updated successfully");

    if (command.restart) {
      console.log("[connector] Restarting...");
      process.exit(0); // launchd/systemd will restart us
    }
  } finally {
    await fs.rm(tmpDir, { recursive: true, force: true });
  }
}
```

---

## Small Examples

### Example 1: Minimal Connector (40 lines)

The simplest possible connector that registers and heartbeats:

```typescript
import WebSocket from "ws";

const HUB = process.env.ATLAS_HUB ?? "wss://api-mom.garywu.dev/connect";
const TOKEN = process.env.ATLAS_TOKEN!;
const MACHINE = process.env.HOSTNAME ?? "unknown";

function connect() {
  const ws = new WebSocket(`${HUB}?machine=${MACHINE}`, {
    headers: { Authorization: `Bearer ${TOKEN}` },
  });

  ws.on("open", () => {
    console.log("Connected");
    ws.send(JSON.stringify({ type: "status-report", machine: MACHINE, busy: false }));
  });

  ws.on("message", (raw) => {
    const msg = JSON.parse(raw.toString());
    if (msg.type === "ping") ws.send(JSON.stringify({ type: "pong" }));
    if (msg.type === "request") {
      ws.send(JSON.stringify({
        type: "response", id: msg.id, status: 200,
        body: { echo: msg.body },
      }));
    }
  });

  ws.on("close", () => setTimeout(connect, 5000));
  ws.on("error", () => {});
}

connect();

// Heartbeat every 60s
setInterval(() => {
  fetch(`${HUB.replace("wss", "https").replace("/connect", "")}/v1/machines/heartbeat`, {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${TOKEN}` },
    body: JSON.stringify({ machine_id: MACHINE, capabilities: ["echo"] }),
  }).catch(() => {});
}, 60_000);
```

### Example 2: GPU Capability Auto-Detection

Detect GPU availability and report it as a capability:

```typescript
import { execFile } from "node:child_process";

interface GpuInfo {
  name: string;
  memoryMb: number;
  freeMemoryMb: number;
  utilization: number;
}

async function detectGpus(): Promise<GpuInfo[]> {
  // Try nvidia-smi first
  try {
    return await detectNvidiaGpus();
  } catch {}

  // Try macOS Metal (Apple Silicon)
  try {
    return await detectAppleGpu();
  } catch {}

  return [];
}

function detectNvidiaGpus(): Promise<GpuInfo[]> {
  return new Promise((resolve, reject) => {
    execFile(
      "nvidia-smi",
      ["--query-gpu=name,memory.total,memory.free,utilization.gpu",
       "--format=csv,noheader,nounits"],
      { timeout: 5000 },
      (err, stdout) => {
        if (err) return reject(err);
        const gpus = stdout.trim().split("\n").map((line) => {
          const [name, totalMem, freeMem, util] = line.split(", ");
          return {
            name: name.trim(),
            memoryMb: parseInt(totalMem),
            freeMemoryMb: parseInt(freeMem),
            utilization: parseInt(util),
          };
        });
        resolve(gpus);
      }
    );
  });
}

function detectAppleGpu(): Promise<GpuInfo[]> {
  return new Promise((resolve, reject) => {
    execFile(
      "system_profiler",
      ["SPDisplaysDataType", "-json"],
      { timeout: 5000 },
      (err, stdout) => {
        if (err) return reject(err);
        const data = JSON.parse(stdout);
        const displays = data.SPDisplaysDataType ?? [];
        const gpus = displays.map((d: Record<string, string>) => ({
          name: d.sppci_model ?? "Apple GPU",
          memoryMb: parseInt(d.sppci_vram ?? "0") * 1024, // Convert GB to MB
          freeMemoryMb: 0, // macOS doesn't expose free VRAM easily
          utilization: 0,
        }));
        resolve(gpus);
      }
    );
  });
}

// Use in capability registration
const gpus = await detectGpus();
if (gpus.length > 0) {
  capabilities.push({
    name: "gpu-inference",
    description: `GPU inference (${gpus[0].name})`,
    resources: { gpu: true },
    tags: gpus.map((g) => g.name),
  });
}
```

### Example 3: mDNS Peer Discovery with Auto-Registration

Find Atlas connectors on the local network and register them with the hub:

```typescript
import { Bonjour } from "bonjour-service";

const bonjour = new Bonjour();

// Advertise ourselves
bonjour.publish({
  name: `atlas-${machineId}`,
  type: "atlas-connector",
  protocol: "tcp",
  port: 7654,
  txt: {
    id: machineId,
    role: "node",
    caps: capabilities.map((c) => c.name).join(","),
    version: "0.1.0",
  },
});

// Discover and report peers to the hub
bonjour.find({ type: "atlas-connector" }, (service) => {
  const txt = service.txt as Record<string, string>;
  if (txt.id === machineId) return; // Skip self

  console.log(`Discovered peer: ${txt.id} at ${service.host}:${service.port}`);
  console.log(`  Capabilities: ${txt.caps}`);

  // Optionally: establish direct connection for P2P communication
  // This is useful for large data transfers (don't route through the hub)
});
```

### Example 4: Heartbeat with Adaptive Interval

Increase heartbeat frequency when the machine is busy, decrease when idle:

```typescript
class AdaptiveHeartbeat {
  private baseInterval: number;
  private minInterval: number;
  private maxInterval: number;
  private currentInterval: number;
  private timer: ReturnType<typeof setTimeout> | null = null;
  private busy = false;

  constructor(options?: {
    baseIntervalMs?: number;
    minIntervalMs?: number;
    maxIntervalMs?: number;
  }) {
    this.baseInterval = options?.baseIntervalMs ?? 60_000;
    this.minInterval = options?.minIntervalMs ?? 10_000;
    this.maxInterval = options?.maxIntervalMs ?? 300_000;
    this.currentInterval = this.baseInterval;
  }

  start(sendHeartbeat: () => Promise<void>): void {
    const tick = async () => {
      await sendHeartbeat();
      this.timer = setTimeout(tick, this.currentInterval);
    };
    tick();
  }

  setBusy(busy: boolean): void {
    const wasBusy = this.busy;
    this.busy = busy;

    if (busy && !wasBusy) {
      // Machine just became busy — heartbeat faster
      this.currentInterval = this.minInterval;
    } else if (!busy && wasBusy) {
      // Machine just became idle — return to base
      this.currentInterval = this.baseInterval;
    }
  }

  /**
   * Gradually increase interval during long idle periods.
   * Called by the heartbeat response handler.
   */
  adjustForIdle(): void {
    if (!this.busy) {
      this.currentInterval = Math.min(
        this.currentInterval * 1.5,
        this.maxInterval
      );
    }
  }

  stop(): void {
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }
  }
}
```

### Example 5: Capability Health Check

Verify that declared capabilities still work:

```typescript
async function healthCheckCapabilities(
  capabilities: Map<string, CapabilityHandler>
): Promise<{ name: string; healthy: boolean; error?: string }[]> {
  const results: { name: string; healthy: boolean; error?: string }[] = [];

  for (const [name, handler] of capabilities) {
    try {
      // Each capability should handle empty/minimal input gracefully
      const result = await Promise.race([
        handler({}),
        new Promise<CapabilityResult>((_, reject) =>
          setTimeout(() => reject(new Error("Health check timeout")), 5000)
        ),
      ]);

      results.push({
        name,
        healthy: result.success || result.error === undefined,
        error: result.error,
      });
    } catch (err) {
      results.push({
        name,
        healthy: false,
        error: err instanceof Error ? err.message : String(err),
      });
    }
  }

  return results;
}

// Run health checks periodically and update capability advertisements
setInterval(async () => {
  const health = await healthCheckCapabilities(CAPABILITY_HANDLERS);
  const unhealthy = health.filter((h) => !h.healthy);

  if (unhealthy.length > 0) {
    console.warn(
      `[connector] Unhealthy capabilities: ${unhealthy.map((h) => h.name).join(", ")}`
    );

    // Re-advertise only healthy capabilities
    const healthyCaps = health
      .filter((h) => h.healthy)
      .map((h) => h.name);

    ws?.send(
      JSON.stringify({
        type: "capability-advertisement",
        capabilities: healthyCaps.map((name) => ({
          name,
          spec: { description: `Capability: ${name}` },
        })),
      })
    );
  }
}, 5 * 60_000); // Every 5 minutes
```

### Example 6: Pipeline Definition as JSON

Define pipelines in a JSON file that the hub loads:

```json
{
  "pipelines": [
    {
      "id": "daily-brief",
      "name": "Daily Brief Pipeline",
      "description": "Gather metrics, summarize, and push to HUD",
      "schedule": "0 8 * * *",
      "steps": [
        {
          "capability": "gather-metrics",
          "timeoutMs": 30000,
          "retryPolicy": { "maxRetries": 2, "backoffMs": 5000 }
        },
        {
          "capability": "llm-summarize",
          "selector": { "matchLabels": { "atlas.dev/inference": "ollama" } },
          "timeoutMs": 60000
        },
        {
          "capability": "hud-notify",
          "selector": { "matchLabels": { "atlas.dev/display": "true" } },
          "inputTransform": "$.summary -> { severity: 'green', message: $.text }"
        }
      ]
    },
    {
      "id": "deploy-preview",
      "name": "Deploy Preview",
      "description": "Build, test, and deploy a preview environment",
      "steps": [
        {
          "capability": "git-clone",
          "timeoutMs": 60000
        },
        {
          "capability": "npm-build",
          "selector": { "matchLabels": { "atlas.dev/arch": "arm64" } },
          "timeoutMs": 300000
        },
        {
          "capability": "wrangler-deploy",
          "timeoutMs": 60000,
          "retryPolicy": { "maxRetries": 3, "backoffMs": 10000 }
        }
      ]
    }
  ]
}
```

### Example 7: WebSocket Message Router

A typed message router that dispatches to handlers based on message type:

```typescript
type MessageHandler<T> = (msg: T) => Promise<void> | void;

class MessageRouter {
  private handlers: Map<string, MessageHandler<any>> = new Map();

  on<T extends { type: string }>(
    type: T["type"],
    handler: MessageHandler<T>
  ): this {
    this.handlers.set(type, handler);
    return this;
  }

  async route(raw: string): Promise<void> {
    let msg: { type: string };
    try {
      msg = JSON.parse(raw);
    } catch {
      console.warn("Unparseable message");
      return;
    }

    const handler = this.handlers.get(msg.type);
    if (handler) {
      await handler(msg);
    } else {
      console.warn(`No handler for message type: ${msg.type}`);
    }
  }
}

// Usage
const router = new MessageRouter()
  .on<PingMessage>("ping", () => {
    ws.send(JSON.stringify({ type: "pong" }));
  })
  .on<DispatchRequest>("request", async (msg) => {
    await handleDispatch(ws, msg, machineId);
  })
  .on<UpdateCommand>("update", async (msg) => {
    await handleUpdate(msg);
  })
  .on<ConfigPush>("config-push", (msg) => {
    applyConfig(msg.config);
  });

ws.on("message", (raw) => router.route(raw.toString()));
```

### Example 8: Graceful Shutdown with Drain

Complete in-flight tasks before shutting down:

```typescript
class GracefulShutdown {
  private activeTasks: Map<string, Promise<void>> = new Map();
  private shutdownRequested = false;
  private drainTimeoutMs: number;

  constructor(drainTimeoutMs = 30_000) {
    this.drainTimeoutMs = drainTimeoutMs;
  }

  /**
   * Track a task. Returns a function to call when the task completes.
   */
  trackTask(taskId: string, promise: Promise<void>): void {
    this.activeTasks.set(taskId, promise);
    promise.finally(() => this.activeTasks.delete(taskId));
  }

  isShuttingDown(): boolean {
    return this.shutdownRequested;
  }

  /**
   * Begin graceful shutdown:
   * 1. Stop accepting new tasks
   * 2. Wait for active tasks to complete (up to drainTimeout)
   * 3. Send deregistration
   * 4. Exit
   */
  async shutdown(
    ws: WebSocket | null,
    config: { apiMomUrl: string; machineId: string; secret: string }
  ): Promise<void> {
    this.shutdownRequested = true;
    console.log(
      `[shutdown] Draining ${this.activeTasks.size} active tasks...`
    );

    // Wait for active tasks with timeout
    const drainPromise = Promise.all(this.activeTasks.values());
    const timeoutPromise = new Promise<void>((resolve) =>
      setTimeout(() => {
        console.warn("[shutdown] Drain timeout — forcing shutdown");
        resolve();
      }, this.drainTimeoutMs)
    );

    await Promise.race([drainPromise, timeoutPromise]);

    // Send final status
    if (ws?.readyState === WebSocket.OPEN) {
      ws.send(
        JSON.stringify({
          type: "status-report",
          machine: config.machineId,
          busy: false,
          version: "0.1.0",
        })
      );
      ws.close(1000, "shutdown");
    }

    // Deregister
    try {
      await fetch(
        `${config.apiMomUrl}/v1/machines/${config.machineId}`,
        {
          method: "DELETE",
          headers: { Authorization: `Bearer ${config.secret}` },
        }
      );
    } catch {
      // Best effort
    }

    console.log("[shutdown] Complete");
    process.exit(0);
  }
}

// Wire up
const shutdown = new GracefulShutdown(30_000);

process.on("SIGINT", () => shutdown.shutdown(ws, config));
process.on("SIGTERM", () => shutdown.shutdown(ws, config));
```

---

## Real-World Multi-Machine AI Systems

### Ray Clusters

[Ray](https://docs.ray.io/en/latest/cluster/getting-started.html) is the most widely adopted framework for distributed AI workloads. A Ray cluster has a head node and worker nodes.

**Architecture:**
- Head node runs the Global Control Store (GCS), scheduler, and API server
- Worker nodes register with the head via `ray start --address=<head>:6379`
- Tasks are scheduled based on resource requirements (CPU, GPU, memory)
- Object store enables zero-copy data sharing between tasks on the same node

**Machine registration:**
```bash
# Start head node
ray start --head --port=6379

# Join worker node
ray start --address="head-ip:6379" --num-cpus=8 --num-gpus=2 \
  --resources='{"custom_resource": 1}'
```

**What Atlas can learn from Ray:**
- Custom resources are arbitrary labels — exactly like Atlas capabilities
- The scheduler does bin-packing (fit multiple tasks on one node if resources allow)
- Autoscaling: Ray can add/remove nodes based on demand
- Ray's `@ray.remote(num_gpus=1)` decorator is the simplest possible capability declaration

### Dask Distributed

[Dask](https://distributed.dask.org/) is a parallel computing library with a central scheduler and distributed workers.

**Key patterns:**
- [Worker resources](https://distributed.dask.org/en/stable/resources.html) are completely abstract — Dask doesn't know what a "GPU" is, you just declare it
- Workers register with the scheduler automatically on startup
- A Nanny process supervises each worker, restarting it on crash
- The scheduler tracks data locality — it tries to run tasks where their input data already lives

```python
# Start scheduler
dask scheduler

# Start worker with custom resources
dask worker tcp://scheduler:8786 --resources "GPU=2" --nthreads 8

# Submit work that requires GPU
client = Client("tcp://scheduler:8786")
future = client.submit(gpu_task, data, resources={"GPU": 1})
```

**What Atlas can learn from Dask:**
- The Nanny pattern: a supervisor process that restarts crashed workers
- Data locality awareness: route tasks to where the data is, not just where the capability is
- Abstract resources: don't hardcode what GPU means, let the user define it

### vLLM Distributed Inference

[vLLM](https://docs.vllm.ai/en/stable/serving/parallelism_scaling/) distributes large language model inference across multiple GPUs and nodes.

**Architecture:**
- Uses Ray for multi-node orchestration
- Tensor parallelism within a node (splits model layers across GPUs)
- Pipeline parallelism across nodes (each node handles different pipeline stages)
- Common pattern: `--tensor-parallel-size=8 --pipeline-parallel-size=2` for 2 nodes with 8 GPUs each

**What Atlas can learn from vLLM:**
- Pipeline parallelism is a real-world Chain pattern — stages of a model run on different machines
- Node homogeneity matters — all nodes must have identical environments for model sharding
- The distinction between TP (within node) and PP (across nodes) maps to "local parallelism" vs "distributed pipeline"

### Ollama + Hive

[Ollama](https://ollama.com/) is primarily a single-machine tool, but the [Hive framework](https://www.sciencedirect.com/science/article/pii/S2352711025001505) adds multi-machine coordination.

**Hive Architecture:**
- **HiveCore:** Central proxy handling client requests, auth, and task queuing
- **HiveNode:** Lightweight agent running alongside Ollama on each machine
- Nodes connect to HiveCore via WireGuard tunnel (no public exposure)
- Load balancing based on GPU memory availability

**What Atlas can learn from Hive:**
- The HiveCore/HiveNode split maps directly to Atlas Hub/Node
- WireGuard for secure connectivity between scattered machines
- GPU memory as the primary routing metric for inference workloads

### Temporal.io

[Temporal](https://temporal.io/) is a workflow orchestration platform where workers poll task queues.

**Key patterns:**
- [Task queues](https://docs.temporal.io/task-queue) are created on demand — no pre-registration needed
- Workers subscribe to queues and process tasks — the server never pushes (unlike Atlas's WebSocket push)
- [Task routing](https://docs.temporal.io/task-routing) pairs specific queues with specific workers for sticky sessions
- Worker versioning: deploy new code without breaking in-flight workflows

**What Atlas can learn from Temporal:**
- Pull vs push: Temporal workers pull tasks, Atlas pushes via WebSocket. Pull is simpler but adds latency. Push is faster but requires maintaining WebSocket connections.
- Sticky sessions: for multi-step tasks that need local state, route all steps to the same machine
- Worker versioning: when you update a connector, new tasks go to new-version workers while in-flight tasks finish on old-version workers

---

## Comparisons

### Multi-Machine Coordination Systems

| System | Discovery | Work Distribution | State | Transport | Best For |
|--------|-----------|-------------------|-------|-----------|----------|
| **Atlas** (Node/Hub/Chain) | Hub registry + mDNS | Push via WebSocket | Durable Objects | WebSocket + HTTPS | AI agent orchestration, mixed cloud/local |
| **Ray** | GCS (head node) | Scheduler push | Distributed object store | gRPC | ML training, distributed inference |
| **Dask** | Scheduler registration | Scheduler push | Distributed memory | TCP | Data science, array computing |
| **Temporal** | Task queue subscription | Worker pull | Persistence layer (Cassandra/PG) | gRPC | Durable workflows, business logic |
| **Consul** | Gossip + DNS | Service mesh routing | Raft consensus KV | HTTP/DNS/gRPC | Service discovery, config management |
| **Kubernetes** | API server + etcd | Scheduler placement | etcd | API server | Container orchestration |

### Install Script Patterns

| Installer | OS Detection | Service Config | Auth Setup | Binary Verification | Uninstall |
|-----------|-------------|----------------|------------|--------------------|-----------|
| **k3s** | uname + /etc/os-release | systemd/openrc auto-detect | K3S_TOKEN env var | SHA256 checksum | k3s-uninstall.sh |
| **Tailscale** | curl + package manager | Native packages (apt/yum) | tailscale login (OAuth) | Package signatures | Package manager |
| **Homebrew** | xcode-select detection | N/A (user-space) | N/A | Git-based (source) | brew uninstall |
| **mise** | uname -s/-m | N/A (user-space) | N/A | Checksum + GPG | mise uninstall |
| **Atlas** (proposed) | uname + init system | launchd/systemd/openrc | Bearer token | SHA256 checksum | --uninstall flag |

### Capability Discovery Approaches

| Approach | Scope | Latency | Fault Tolerance | Complexity | Best For |
|----------|-------|---------|-----------------|------------|----------|
| **Central registry** (Consul/etcd) | Global | Sub-ms | Depends on registry HA | Medium | Cross-datacenter services |
| **Gossip** (Serf/SWIM) | Cluster | O(log N) rounds | Very high (decentralized) | High | Large clusters, partition-tolerant |
| **mDNS/Bonjour** | LAN | Sub-second | High (no central point) | Low | Local network, zero-config |
| **K8s labels + scheduler** | Cluster | Sub-ms | API server HA | Medium | Container workloads |
| **Hub-spoke** (Atlas) | Global | Sub-ms | Hub is SPoF | Low | Simple topologies, AI agents |
| **Tailscale ACLs** | Global mesh | Sub-ms | Coordination server HA | Low (outsourced) | Secure cross-network |

### Pipeline / Workflow Engines

| Engine | Definition | Routing | Checkpointing | Fan-Out | Language |
|--------|-----------|---------|---------------|---------|----------|
| **Atlas Chain** | JSON declarative | Capability-based | DO storage | Manual (code) | TypeScript |
| **Temporal** | Code (SDK) | Task queue | Built-in (server) | Built-in (child workflows) | Multi-language |
| **Airflow** | Python DAG | Executor pool | Task instance state | Built-in (dynamic tasks) | Python |
| **Prefect** | Python flow | Work pool | Built-in (Prefect Cloud) | Built-in (.map()) | Python |
| **N8N** | Visual (JSON) | Node connection | Execution table | Built-in (split node) | TypeScript |
| **Node-RED** | Visual (JSON) | Wire connections | None (stateless) | Built-in (split node) | JavaScript |

---

## Anti-Patterns

| Don't | Do Instead | Why |
|-------|-----------|-----|
| Hardcode machine addresses in pipeline definitions | Use capability names and let the hub route | Machines come and go. IP addresses change. Capabilities are the stable abstraction. |
| Use a single shared token for all machines | Issue per-machine tokens via registration flow | One leaked token compromises one machine, not the entire network. |
| Skip the heartbeat and rely on WebSocket connection state | Heartbeat independently of WebSocket | WebSocket can appear connected (TCP half-open) while the machine is actually dead. Heartbeats confirm liveness at the application level. |
| Put secrets in the install script URL | Pass secrets via environment variables or stdin | `curl https://install.example.com?token=SECRET` shows up in shell history, server logs, and proxy logs. |
| Restart failed pipelines from step 0 | Checkpoint after each step and resume from the last checkpoint | Re-executing expensive steps (GPU inference, large downloads) wastes time and money. |
| Register capabilities once at startup | Re-evaluate capabilities at each heartbeat | GPU VRAM fills up. Disk space runs out. Services crash. Capabilities are dynamic, not static. |
| Route all traffic through the hub | Use hub for coordination but allow P2P for data transfer | The hub becomes a bottleneck for large payloads. Machines on the same LAN should transfer data directly. |
| Reconnect immediately after disconnect | Exponential backoff with jitter | Thundering herd: all machines reconnect at once and crash the hub again. |
| Deploy updates to all machines at once | Rolling update (10-20% at a time) | If the update is broken, you lose your entire fleet instead of 10-20%. |
| Block the main thread during capability execution | Run handlers in worker threads or async | A long-running capability blocks heartbeats, causing the hub to think the machine is dead. |
| Use `hostname` as the machine ID | Generate a UUID and persist it to disk | Hostnames can change. Two machines can have the same hostname. UUIDs are globally unique and stable. |
| Skip the uninstall script | Always provide a clean uninstall path | Users who can't cleanly remove your software won't install it in the first place. |

---

## References

### Official Documentation

- [K3s Installation](https://docs.k3s.io/installation/configuration) — K3s install script documentation, environment variables, and configuration options
- [K3s Install Script Source](https://github.com/k3s-io/k3s/blob/main/install.sh) — The actual k3s install.sh, gold standard for `curl | sh` installers
- [K3s Quick Start Guide](https://docs.k3s.io/quick-start) — Getting started with K3s in under 5 minutes
- [Consul Service Discovery](https://developer.hashicorp.com/consul/docs/use-case/service-discovery) — How Consul handles service registration, health checks, and DNS-based discovery
- [Consul Service Registration Tutorial](https://developer.hashicorp.com/consul/tutorials/get-started-vms/virtual-machine-gs-service-discovery) — Step-by-step guide to registering services with Consul agents
- [Temporal Task Queues](https://docs.temporal.io/task-queue) — How Temporal routes work to workers via task queues
- [Temporal Task Routing](https://docs.temporal.io/task-routing) — Pairing task queues with specific workers for capability-based routing
- [Temporal Workers](https://docs.temporal.io/workers) — What a Temporal worker is and how it registers with the server
- [Kubernetes Node Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) — Labels, selectors, and affinity rules for scheduling pods on specific nodes
- [Kubernetes Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) — How nodes repel pods and how pods opt-in to tainted nodes
- [Kubernetes Well-Known Labels](https://kubernetes.io/docs/reference/labels-annotations-taints/) — Standard label keys for topology, architecture, OS, and instance type
- [Ray Clusters Overview](https://docs.ray.io/en/latest/cluster/getting-started.html) — How to set up and manage Ray clusters for distributed AI
- [Dask Worker Resources](https://distributed.dask.org/en/stable/resources.html) — Abstract resource declarations for Dask workers
- [Dask Worker Documentation](https://distributed.dask.org/en/stable/worker.html) — Worker lifecycle, nanny process, and resource management
- [Dask Scheduling Policies](https://distributed.dask.org/en/latest/scheduling-policies.html) — How the Dask scheduler makes task placement decisions
- [vLLM Parallelism and Scaling](https://docs.vllm.ai/en/stable/serving/parallelism_scaling/) — Tensor parallel, pipeline parallel, and multi-node inference with vLLM
- [vLLM Distributed Serving](https://docs.vllm.ai/en/v0.8.0/serving/distributed_serving.html) — Multi-node distributed inference setup
- [Tailscale: How it Works](https://tailscale.com/blog/how-tailscale-works) — Deep dive into Tailscale's WireGuard-based mesh networking and coordination server
- [Tailscale: What is Tailscale](https://tailscale.com/kb/1151/what-is-tailscale) — Overview of Tailscale's zero-config VPN

### Libraries and Tools

- [bonjour-service](https://github.com/node-opcua/bonjour-service) — TypeScript Bonjour/Zeroconf protocol implementation for mDNS service discovery in Node.js
- [node_mdns](https://github.com/agnat/node_mdns) — Native Node.js addon for mDNS/Zeroconf/Bonjour service discovery
- [dns-sd (earthstar)](https://github.com/earthstar-project/dns-sd) — DNS-SD implementation in TypeScript for Deno and Node (RFC 6763 compliant)
- [ws (WebSocket)](https://www.npmjs.com/package/ws) — Simple, fast WebSocket implementation for Node.js
- [Zod](https://zod.dev/) — TypeScript-first schema validation for capability input validation

### Blog Posts and Guides

- [Ray Clusters for AI: Distributed Computing Architecture](https://introl.com/blog/ray-clusters-distributed-ai-computing-infrastructure-guide-2025) — 2025 guide to Ray cluster architecture for AI workloads
- [Building a Distributed AI System with Ray and vLLM on Mac Minis](https://www.doppler.com/blog/building-a-distributed-ai-system-how-to-set-up-ray-and-vllm-on-mac-minis) — Practical guide to distributed AI inference on Apple Silicon
- [Distributed Inference with vLLM](https://blog.vllm.ai/2025/02/17/distributed-inference.html) — Official vLLM blog post on multi-node inference strategies
- [Hive: A Secure, Scalable Framework for Distributed Ollama Inference](https://www.sciencedirect.com/science/article/pii/S2352711025001505) — Academic paper on the Hive framework for distributed Ollama
- [Running Local LLMs with Ollama: 3 Levels](https://www.bentoml.com/blog/running-local-llms-with-ollama-3-levels-from-local-to-distributed-inference) — From single machine to distributed cluster with Ollama
- [Advanced Service Discovery: Consul, Etcd, and Zookeeper](https://ahmettsoner.medium.com/advanced-service-discovery-in-microservices-consul-etcd-and-zookeeper-b8860dce8363) — Comparison of service discovery approaches
- [Temporal Internal Architecture Breakdown](https://medium.com/data-science-collective/system-design-series-a-step-by-step-breakdown-of-temporals-internal-architecture-52340cc36f30) — Deep dive into Temporal's internal system design

### RFCs and Standards

- [RFC 6762: Multicast DNS](https://datatracker.ietf.org/doc/html/rfc6762) — The mDNS protocol specification
- [RFC 6763: DNS-Based Service Discovery](https://datatracker.ietf.org/doc/html/rfc6763) — DNS-SD protocol for discovering services on local networks
- [WireGuard Protocol](https://www.wireguard.com/protocol/) — The cryptographic protocol underlying Tailscale

### Security

- [Bash Scripting Best Practices for Reliable Automation](https://oneuptime.com/blog/post/2026-02-13-bash-best-practices/view) — Security and reliability patterns for shell scripts
- [Shell Script Security (Apple)](https://developer.apple.com/library/archive/documentation/OpenSource/Conceptual/ShellScripting/ShellScriptSecurity/ShellScriptSecurity.html) — Apple's guide to writing secure shell scripts
