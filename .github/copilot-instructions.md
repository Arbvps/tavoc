---
name: tavoc
description: >-
  Cloudflare Containers starter demonstrating TypeScript Workers orchestrating
  Go containers via Durable Objects. Multi-language full-stack project with
  container lifecycle management, load balancing, and error handling.
---

# Tavoc: Cloudflare Containers Starter

## Getting Started (New Developers)

### Quick Setup Checklist
- [ ] Node.js 18+ installed (`node --version`)
- [ ] Go 1.24+ installed (`go version`)
- [ ] Docker installed (for local container builds)
- [ ] Wrangler authenticated (`wrangler login` or set `CLOUDFLARE_API_TOKEN`)
- [ ] Run `npm install`
- [ ] Run `npm run dev` — dev server appears at `http://localhost:8787`
- [ ] Test: Visit `http://localhost:8787/container/test-1` — should see container response

**Stuck?** See [Debugging](#debugging--troubleshooting) section below.

## Project Overview

**Tavoc** is a multi-language starter showcasing **Cloudflare Containers** + **Workers** + **Durable Objects**:

- **Frontend/Orchestrator**: TypeScript Worker (src/index.ts) using Hono — routes requests, manages container lifecycle
- **Backend Compute**: Go HTTP server (container_src/) — runs on port 8080 inside containerized Durable Objects
- **Runtime**: Cloudflare Workers (TypeScript), Containers (Go), Durable Objects (state/lifecycle)
- **Key patterns**: per-ID container routing, load balancing, lifecycle hooks (onStart, onStop, onError)

## Build & Development Commands

| Command | Purpose |
|---------|---------|
| `npm install` | Install Node dependencies (Hono, wrangler, TypeScript) |
| `npm run dev` | Start dev server on **localhost:8787** (watch mode, hot reload for src/) |
| `npm run deploy` | Deploy to Cloudflare production (requires auth) |
| `npm run cf-typegen` | Generate Cloudflare Durable Objects types |

**Dev environment:**
- Worker accessible at `http://localhost:8787/*`
- Containers run locally at `http://localhost:8080` (proxied through Worker)
- Go server logs appear in terminal running `npm run dev`
- Changes to src/ auto-reload; container_src/ changes require manual restart

## Directory Structure & Component Responsibilities

```
tavoc/
├── src/
│   └── index.ts                 # TypeScript Worker (Hono app)
├── container_src/
│   ├── main.go                  # Go HTTP server (port 8080)
│   └── go.mod                   # Go module definition
├── Dockerfile                   # Multi-stage build (Alpine → scratch)
├── wrangler.jsonc               # Cloudflare config (bindings, env, migrations)
├── tsconfig.json                # TypeScript config (strict: true)
├── package.json                 # Node dependencies
└── worker-configuration.d.ts    # Auto-generated Durable Objects types
```

### src/index.ts — TypeScript Worker
**Purpose:** HTTP request orchestrator; manages container lifecycle via Durable Objects.

**Key responsibilities:**
- Route incoming requests to containers or load-balancer
- Instantiate containers by ID (Durable Objects)
- Inject environment variables into container process
- Handle errors and graceful shutdowns

**Routing examples:**
```
GET /container/my-app-1      → Launch/access container with ID "my-app-1"
GET /lb?id=lb-test           → Load-balance across multiple instances
GET /singleton               → Single persistent instance
GET /error                   → Trigger onError hook (panic demo)
```

### container_src/main.go — Go HTTP Server
**Purpose:** Compute workload running inside Durable Object containers.

**Key responsibilities:**
- Listen on `:8080` for HTTP requests
- Handle your business logic (fast, CPU-bound tasks)
- Graceful shutdown on signals
- Optional lifecycle hooks: `onStart()`, `onStop()`, `onError()`

**Example server:**
```go
package main
import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        msg := os.Getenv("MESSAGE") // From Cloudflare config
        fmt.Fprintf(w, "Hello from container: %s\n", msg)
    })
    http.ListenAndServe(":8080", nil)
}
```

### Dockerfile — Multi-Stage Build
**Purpose:** Compile Go binary in Alpine (large), copy to scratch (minimal).

**Two-stage approach:**
1. **Builder stage** (Alpine): Downloads Go deps, compiles binary
2. **Runtime stage** (scratch): Contains only the binary; minimal footprint

**Key optimization:**
```dockerfile
# Stage 1: Build (Alpine)
FROM alpine:latest AS builder
WORKDIR /build
COPY container_src/go.mod go.mod
RUN go mod download  # Cache layer
COPY container_src/ .
RUN CGO_ENABLED=0 GOOS=linux go build -o app main.go

# Stage 2: Runtime (scratch)
FROM scratch
COPY --from=builder /build/app /app
ENTRYPOINT ["/app"]
```

### wrangler.jsonc — Cloudflare Configuration
**Purpose:** Define Worker, Durable Objects, environment variables, deployment settings.

**Key sections:**
```jsonc
{
  "compatibility_date": "2024-01-01",
  "main": "src/index.ts",
  "env": {
    "production": {
      "route": "example.com/api/*",
      "zone_id": "YOUR_ZONE_ID"
    }
  },
  "env_vars": {
    "MESSAGE": "Production message"
  },
  "durable_objects": {
    "bindings": [{
      "name": "CONTAINER",
      "class_name": "Container"
    }]
  }
}
```

## Development Conventions

### 1. Container IDs & Routing

Each container is identified by a unique ID (string). Routes create or reuse containers based on ID:

| Route | Behavior | Example |
|-------|----------|---------|
| `/container/:id` | Launch or reuse container with ID `:id` | `GET /container/user-123` → container persists until inactive timeout |
| `/lb?id=name` | Load-balance across multiple instances of same ID | Multiple requests spawn multiple containers; requests distributed |
| `/singleton` | Single persistent instance (no ID) | Useful for shared state or background jobs |
| `/error` | Deliberately trigger container crash (`panic`) | Tests `onError()` hook and recovery behavior |

**Example workflow:**
```
1. Request: GET /container/api-task-1
2. Worker: Does container "api-task-1" exist? 
   - NO → Spin up new Durable Object, start container
   - YES → Reuse existing instance
3. Container: Receives HTTP request on :8080, processes, returns response
4. Worker: Proxies response back to client
```

### 2. Environment Variables

Variables injected into container process at startup. Define in `wrangler.jsonc`:

```jsonc
{
  "env_vars": {
    "MESSAGE": "Hello from Cloudflare",
    "LOG_LEVEL": "debug",
    "API_ENDPOINT": "https://api.example.com"
  }
}
```

**Access in Go:**
```go
package main
import "os"

msg := os.Getenv("MESSAGE")
logLevel := os.Getenv("LOG_LEVEL")
```

**Auto-injected:**
- `CLOUDFLARE_DURABLE_OBJECT_ID` — Unique ID assigned to this container instance

### 3. Lifecycle Hooks

Optional functions in your Go server to respond to container events:

```go
// Called when container starts
func onStart(containerID string, env map[string]string) {
    log.Println("Container started:", containerID)
    // Initialize resources, connect to databases, etc.
}

// Called when container gracefully stops
func onStop(containerID string) {
    log.Println("Container stopping:", containerID)
    // Cleanup: close connections, flush logs, etc.
}

// Called on panic or error
func onError(containerID string, err error) {
    log.Println("Container error:", containerID, err)
    // Log to external system, trigger alerts, etc.
}
```

### 4. TypeScript Strictness

All TypeScript files must comply with `strict: true` in tsconfig.json:

```typescript
// ✅ GOOD: Explicit types
function handleRequest(req: Request): Response {
  const id: string = req.url.split("/").pop() || "default";
  return new Response(id);
}

// ❌ BAD: Missing types (strict mode fails)
function handleRequest(req) {  // req: any ← error
  const id = req.url.split("/").pop();  // id: any ← error
  return new Response(id);
}
```

## Key Files & Patterns

- **wrangler.jsonc**: Container image link, Durable Objects binding, environment config, migrations
- **Dockerfile**: Alpine-based Go builder; use `go mod download` to cache dependencies
- **worker-configuration.d.ts**: Custom types for Worker environment (auto-generated)
- **package-lock.json**: Locked dependency versions for reproducibility

## Debugging & Troubleshooting

### Common Issues & Solutions

| Issue | Diagnosis | Solution |
|-------|-----------|----------|
| Container doesn't start | Check `npm run dev` logs; look for Go build errors | 1. Run `go mod tidy` in container_src/ <br> 2. Ensure `go mod download` succeeds in Dockerfile <br> 3. Verify port 8080 is free: `netstat -ano \| findstr :8080` (Windows) or `lsof -i :8080` (Mac/Linux) |
| Port 8787 already in use | `npm run dev` fails with "port in use" error | Change dev port in `wrangler.jsonc`: `"port": 8788` or kill process: `lsof -i :8787 \| grep -v PID \| awk '{print $2}' \| xargs kill -9` |
| Environment vars not injected | Container doesn't see vars set in wrangler.jsonc | 1. Verify `env_vars` section in wrangler.jsonc is correct <br> 2. Restart `npm run dev` after editing <br> 3. Verify syntax: `wrangler.jsonc` must be valid JSON (watch for trailing commas) |
| Container inactivity timeout (sleeps after 2 min) | Expected behavior | Container auto-restarts on next request; configurable in wrangler.jsonc |
| Types missing in src/ (worker-configuration.d.ts) | IDE can't find Cloudflare types | Run `npm run cf-typegen` to regenerate |
| Go binary too large | Container image size balloons | 1. Verify Dockerfile final stage uses `FROM scratch` (not alpine) <br> 2. Check Dockerfile uses `CGO_ENABLED=0` <br> 3. Strip debug symbols: add `-ldflags="-s -w"` to `go build` |
| TypeScript compilation fails | `npm run dev` shows type errors | 1. Check `tsconfig.json` strict mode settings <br> 2. Add explicit type annotations (especially for function arguments) <br> 3. Run `npm run cf-typegen` if types are from Cloudflare |

### Debugging Strategies

#### View Container Logs
```bash
# Terminal 1: Start dev server
npm run dev

# Terminal 2: Tail logs
tail -f /tmp/wrangler.log  # or equivalent on Windows

# Or: Check stderr in `npm run dev` output directly
```

#### Test Container Directly
```bash
# Request specific container
curl http://localhost:8787/container/debug-1

# Request load-balanced instance
curl http://localhost:8787/lb?id=test-lb

# Trigger error hook (panic)
curl http://localhost:8787/error
```

#### Inspect Go Server Startup
Add logging to `container_src/main.go`:
```go
package main
import (
    "log"
    "net/http"
)

func init() {
    log.Println("Go server initializing...")
    log.Printf("MESSAGE env var: %s\n", os.Getenv("MESSAGE"))
}

func main() {
    log.Println("Starting HTTP server on :8080")
    http.ListenAndServe(":8080", nil)
}
```

#### Local Docker Test
```bash
# Build image locally
docker build -t tavoc-test .

# Run container
docker run -p 8080:8080 -e MESSAGE="test" tavoc-test

# Test
curl http://localhost:8080/
```

#### TypeScript Type Errors
```bash
# Regenerate types
npm run cf-typegen

# Check tsconfig
cat tsconfig.json | grep strict

# Run TypeScript compiler in check-only mode
npx tsc --noEmit
```

## When to Edit Each Component

- **Editing src/index.ts**: Changing request routing, container lifecycle, error handling, or adding new endpoints
- **Editing container_src/main.go**: Changing container logic, HTTP handlers, startup/shutdown behavior
- **Editing Dockerfile**: Changing build process, adding dependencies, or optimizing container size
- **Editing wrangler.jsonc**: Changing environment variables, bindings, or deployment config

## Deployment Workflow

### Pre-Deployment Checklist

1. **Authenticate with Cloudflare**:
   ```bash
   wrangler login
   # or set CLOUDFLARE_API_TOKEN environment variable
   export CLOUDFLARE_API_TOKEN=your_token_here
   ```

2. **Verify types are generated**:
   ```bash
   npm run cf-typegen
   ```

3. **Test locally**:
   ```bash
   npm run dev
   # Visit http://localhost:8787/container/test-1
   # Verify container starts and responds
   ```

4. **Build container image** (Cloudflare will do this, but test locally):
   ```bash
   docker build -t tavoc-prod .
   docker run -p 8080:8080 -e MESSAGE="prod" tavoc-prod
   curl http://localhost:8080/  # Should respond
   ```

### Deploy to Production

```bash
npm run deploy
```

**What happens:**
1. TypeScript compiled and bundled
2. Go binary built and containerized (Dockerfile executed)
3. Container image pushed to Cloudflare Registry
4. Worker script deployed to Cloudflare edge
5. Durable Objects bindings activated

### Environment-Specific Configuration

In `wrangler.jsonc`:
```jsonc
{
  "env": {
    "production": {
      "route": "api.example.com/container/*",
      "zone_id": "YOUR_ZONE_ID",
      "env_vars": {
        "MESSAGE": "Production"
      }
    },
    "staging": {
      "route": "staging-api.example.com/container/*",
      "env_vars": {
        "MESSAGE": "Staging"
      }
    }
  }
}
```

Deploy to specific environment:
```bash
npm run deploy -- --env staging
```

### Monitoring Post-Deployment

```bash
# View logs
wrangler tail

# Check deployment status
wrangler deployments list

# Rollback if needed
wrangler rollback --version <previous_version>
```

## Resources

- [Cloudflare Containers Docs](https://developers.cloudflare.com/containers/)
- [Cloudflare Container Class Repo](https://github.com/cloudflare/containers)
- [Durable Objects Guide](https://developers.cloudflare.com/durable-objects/)
- [Wrangler CLI Reference](https://developers.cloudflare.com/workers/wrangler/)
