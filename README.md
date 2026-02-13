````markdown
# Real-Time Async Service Backend

A high-performance backend infrastructure using raw WebSockets for bidirectional communication, BullMQ for async job queuing, and Bun as the runtime. Designed for long-running services with instant push notifications (no polling).

## Prerequisites

- **Bun** (v1.3+ recommended): Install from [bun.sh](https://bun.sh)
  ```bash
  curl -fsSL https://bun.sh/install | bash
  ```
````

- **Docker**: For running Redis (official image)

## Project Structure (Recommended)

```
project-root/
├── gateway/          # WebSocket gateway (Fastify + @fastify/websocket)
│   ├── package.json
│   └── src/index.ts
├── worker/           # BullMQ workers (separate process)
│   ├── package.json
│   └── src/worker.ts
├── shared/           # Optional: Shared types, utils, message schemas
├── docker-compose.yml
└── .env              # Shared environment variables
```

## Installation

1. Clone the repo and navigate to root.

2. Install dependencies in each package (Bun is used for fast installs):

   ```bash
   # In gateway/
   cd gateway
   bun install

   # In worker/
   cd ../worker
   bun install
   ```

   Key dependencies (add to each `package.json` as needed):

   ```json
   {
     "dependencies": {
       "fastify": "^4.28.0",
       "@fastify/websocket": "^8.3.0",
       "bullmq": "^5.0.0",
       "ioredis": "^5.4.0",
       "zod": "^3.23.0" // Optional: for message validation
     }
   }
   ```

3. Create a shared `.env` file (at project root):
   ```
   REDIS_URL=redis://localhost:6379
   PORT=3000                  # Gateway port
   JWT_SECRET=your-secret      # For auth
   ```

## Running Redis (Docker)

Use the official Redis image via Docker Compose (recommended for consistency):

Create `docker-compose.yml` at root:

```yaml
version: "3.8"
services:
  redis:
    image: redis:latest
    container_name: async-backend-redis
    ports:
      - "6379:6379"
    restart: unless-stopped
```

Start Redis:

```bash
docker compose up -d
```

(Or one-off: `docker run --name async-redis -p 6379:6379 -d redis:latest`)

## Running the Application

1. **Start the Gateway** (handles WebSocket connections):

   ```bash
   cd gateway
   bun run src/index.ts   # Or bun dev if using scripts
   ```

2. **Start Workers** (process jobs – run multiple for scaling):

   ```bash
   cd worker
   bun run src/worker.ts
   ```

   Run additional worker instances in separate terminals for parallelism.

## Development Notes

- Implement heartbeats (ping/pong) manually on WebSocket connections.
- Connection mapping: In-memory `Map<jobId, Set<WebSocket>>` in gateway for targeted pushes.
- Test with sample clients from the [`websocket-backend.md`](./websocket-backend.md)
- Test with samele workers see [`WORKERS.md`](./WORKER.md)

## Questions to ponder

Should this be a monorepo setup (single package.json) or separate packages for gateway/worker as shown?

## NOTE : This specification has been developed/edited jointly by `GROK.4 Thinking` in coversations with Murthy
