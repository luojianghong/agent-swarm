# Deployment Guide

This guide covers all deployment options for Agent Swarm MCP.

## Table of Contents

- [Docker Compose (Recommended)](#docker-compose-recommended)
- [Docker Worker](#docker-worker)
- [Server Deployment (systemd)](#server-deployment-systemd)
- [Environment Variables](#environment-variables)
- [System Prompts](#system-prompts)
- [Service Registry (PM2)](#service-registry-pm2)
- [Publishing (Maintainers)](#publishing-maintainers)

---

## Docker Compose (Recommended)

The easiest way to deploy a full swarm with API, workers, and lead agent.

### Quick Start

```bash
# Copy example files
cp docker-compose.example.yml docker-compose.yml

# Create a combined .env for docker-compose (combines API + worker settings)
# You can merge .env.example (API) and .env.docker.example (worker) settings
cp .env.docker.example .env

# Edit .env with your values
vim .env

# Start the swarm
docker-compose up -d
```

> **Note:** `.env.example` contains API server settings, `.env.docker.example` contains Docker worker settings. For docker-compose, you need both sets of variables in a single `.env` file.

### What's Included

The example `docker-compose.yml` sets up:

- **API service** (port 3013) - MCP HTTP server
- **2 Worker agents** - Containerized Claude workers
- **1 Lead agent** - Coordinator agent

### Configuration

Edit your `.env` file:

```bash
# Required
API_KEY=your-secret-api-key
CLAUDE_CODE_OAUTH_TOKEN=your-oauth-token  # Run `claude setup-token` to get this

# Agent IDs (optional, auto-generated if not set)
AGENT_ID=worker-1-uuid
AGENT_ID_2=worker-2-uuid
AGENT_ID_LEAD=lead-agent-uuid

# GitHub integration (optional)
GITHUB_TOKEN=your-github-token
GITHUB_EMAIL=your@email.com
GITHUB_NAME=Your Name
```

### Volumes

| Volume | Purpose |
|--------|---------|
| `swarm_api` | SQLite database persistence |
| `swarm_logs` | Session logs |
| `swarm_shared` | Shared workspace between agents |
| `swarm_worker_*` | Personal workspace per worker |

---

## Docker Worker

Run individual Claude workers in containers.

### Pull from Registry

```bash
docker pull ghcr.io/desplega-ai/agent-swarm-worker:latest
```

### Build Locally

```bash
# Build the worker image
docker build -f Dockerfile.worker -t agent-swarm-worker .

# Or using npm script
bun run docker:build:worker
```

### Run

```bash
# Using pre-built image
docker run --rm -it \
  -e CLAUDE_CODE_OAUTH_TOKEN=your-token \
  -e API_KEY=your-api-key \
  -v ./logs:/logs \
  -v ./work:/workspace \
  ghcr.io/desplega-ai/agent-swarm-worker

# With custom system prompt
docker run --rm -it \
  -e CLAUDE_CODE_OAUTH_TOKEN=your-token \
  -e API_KEY=your-api-key \
  -e WORKER_SYSTEM_PROMPT="You are a Python specialist" \
  -v ./logs:/logs \
  -v ./work:/workspace \
  ghcr.io/desplega-ai/agent-swarm-worker

# With system prompt from file
docker run --rm -it \
  -e CLAUDE_CODE_OAUTH_TOKEN=your-token \
  -e API_KEY=your-api-key \
  -e WORKER_SYSTEM_PROMPT_FILE=/workspace/prompts/specialist.txt \
  -v ./work:/workspace \
  ghcr.io/desplega-ai/agent-swarm-worker

# Using npm script (requires .env.docker file)
bun run docker:run:worker
```

### Troubleshooting

**Permission denied when writing to /workspace**

```bash
# Option 1: Fix permissions on host directory
chmod 777 ./work

# Option 2: Run container as your current user
docker run --rm -it --user $(id -u):$(id -g) \
  -e CLAUDE_CODE_OAUTH_TOKEN=your-token \
  -e API_KEY=your-api-key \
  -v ./work:/workspace \
  ghcr.io/desplega-ai/agent-swarm-worker

# Option 3: Create the file on the host first
touch ./work/.mcp.json
chmod 666 ./work/.mcp.json
```

### Architecture

The Docker worker image uses a multi-stage build:

1. **Builder stage**: Compiles `src/cli.tsx` into a standalone binary
2. **Runtime stage**: Ubuntu 24.04 with full development environment

**Pre-installed tools:**

- **Languages**: Python 3, Node.js 22, Bun
- **Build tools**: gcc, g++, make, cmake
- **Process manager**: PM2 (for background services)
- **Utilities**: git, git-lfs, vim, nano, jq, curl, wget, ssh
- **Sudo access**: Worker can install packages with `sudo apt-get install`

**Volumes:**

- `/workspace` - Working directory for cloning repos
- `/logs` - Session logs

### Startup Scripts

Run custom initialization before the worker starts. Place a script at `/workspace/start-up.*`:

**Supported formats** (priority order):
- `start-up.sh` / `start-up.bash` - Bash scripts
- `start-up.js` - Node.js scripts
- `start-up.ts` / `start-up.bun` - Bun/TypeScript scripts

**Interpreter detection:**
1. Shebang line (e.g., `#!/usr/bin/env bun`)
2. File extension (`.ts` -> bun, `.js` -> node, `.sh` -> bash)

**Error handling:**
- `STARTUP_SCRIPT_STRICT=true` (default) - Container exits if script fails
- `STARTUP_SCRIPT_STRICT=false` - Logs warning and continues

**Example: Install dependencies**

```bash
#!/bin/bash
# /workspace/start-up.sh

echo "Installing dependencies..."
if [ -f "package.json" ]; then
    bun install
fi

sudo apt-get update -qq
sudo apt-get install -y -qq ripgrep
```

**Example: TypeScript setup**

```typescript
#!/usr/bin/env bun
// /workspace/start-up.ts

console.log("Running startup...");
await Bun.$`bun install`;

if (!process.env.API_KEY) {
  console.error("ERROR: API_KEY not set");
  process.exit(1);
}
```

---

## Server Deployment (systemd)

Deploy the MCP server to a Linux host with systemd.

### Prerequisites

- Linux with systemd
- Bun installed (`curl -fsSL https://bun.sh/install | bash`)

### Install

```bash
git clone https://github.com/desplega-ai/agent-swarm.git
cd agent-swarm
sudo bun deploy/install.ts
```

This will:
- Copy files to `/opt/agent-swarm`
- Create `.env` file (edit to set `API_KEY`)
- Install systemd service with health checks every 30s
- Start the service on port 3013

### Update

```bash
git pull
sudo bun deploy/update.ts
```

### Management

```bash
# Check status
sudo systemctl status agent-swarm

# View logs
sudo journalctl -u agent-swarm -f

# Restart
sudo systemctl restart agent-swarm

# Stop
sudo systemctl stop agent-swarm
```

---

## Environment Variables

### Docker Worker Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `CLAUDE_CODE_OAUTH_TOKEN` | Yes | OAuth token for Claude CLI (run `claude setup-token`) |
| `API_KEY` | Yes | API key for MCP server |
| `AGENT_ID` | No | Agent UUID (assigned on join if not set) |
| `MCP_BASE_URL` | No | MCP server URL (default: `http://host.docker.internal:3013`) |
| `SESSION_ID` | No | Log folder name (auto-generated if not provided) |
| `WORKER_YOLO` | No | Continue on errors (default: `false`) |
| `WORKER_SYSTEM_PROMPT` | No | Custom system prompt text |
| `WORKER_SYSTEM_PROMPT_FILE` | No | Path to system prompt file |
| `STARTUP_SCRIPT_STRICT` | No | Exit on startup script failure (default: `true`) |
| `SWARM_URL` | No | Base domain for service URLs (default: `localhost`) |
| `SERVICE_PORT` | No | Host port for exposed services (default: `3000`) |
| `PM2_HOME` | No | PM2 state directory (default: `/workspace/.pm2`) |

### Server Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `MCP_BASE_URL` | Base URL for the MCP server | `https://agent-swarm-mcp.desplega.sh` |
| `PORT` | Port for self-hosted MCP server | `3013` |
| `API_KEY` | API key for server authentication | - |

---

## System Prompts

Customize Claude's behavior with system prompts for worker and lead agents.

### CLI Usage

```bash
# Inline system prompt
bunx @desplega.ai/agent-swarm worker --system-prompt "You are a Python specialist."

# System prompt from file
bunx @desplega.ai/agent-swarm worker --system-prompt-file ./prompts/python-specialist.txt

# Same options work for lead agent
bunx @desplega.ai/agent-swarm lead --system-prompt "You are a project coordinator."
bunx @desplega.ai/agent-swarm lead --system-prompt-file ./prompts/coordinator.txt
```

### Docker Usage

```bash
# Using inline system prompt
docker run --rm -it \
  -e CLAUDE_CODE_OAUTH_TOKEN=your-token \
  -e API_KEY=your-api-key \
  -e WORKER_SYSTEM_PROMPT="You are a Python specialist." \
  ghcr.io/desplega-ai/agent-swarm-worker

# Using system prompt file
docker run --rm -it \
  -e CLAUDE_CODE_OAUTH_TOKEN=your-token \
  -e API_KEY=your-api-key \
  -e WORKER_SYSTEM_PROMPT_FILE=/workspace/prompts/specialist.txt \
  -v ./work:/workspace \
  ghcr.io/desplega-ai/agent-swarm-worker
```

### Priority

- CLI flags > Environment variables
- Inline text (`*_SYSTEM_PROMPT`) > File (`*_SYSTEM_PROMPT_FILE`)

---

## Service Registry (PM2)

Workers can run background services on port 3000 using PM2. Services are registered for discovery and auto-restart.

### PM2 Commands

```bash
pm2 start /workspace/app/server.js --name my-api  # Start a service
pm2 stop|restart|delete my-api                     # Manage services
pm2 logs [name]                                    # View logs
pm2 list                                           # Show running processes
```

### MCP Tools

- `register-service` - Register service for discovery and auto-restart
- `unregister-service` - Remove from registry
- `list-services` - Find services exposed by other agents
- `update-service-status` - Update health status

### Starting a New Service

```bash
# 1. Start your service with PM2
pm2 start /workspace/myapp/server.js --name my-api

# 2. Register it (via MCP tool)
# register-service name="my-api" script="/workspace/myapp/server.js"

# 3. Mark healthy when ready
# update-service-status name="my-api" status="healthy"
```

### Service URL Pattern

`https://{service-name}.{SWARM_URL}`

### Health Checks

Implement a `/health` endpoint returning 200 OK for monitoring.

---

## Publishing (Maintainers)

```bash
# Requires gh CLI authenticated
bun deploy/docker-push.ts
```

This builds, tags with version from package.json + `latest`, and pushes to GHCR.
