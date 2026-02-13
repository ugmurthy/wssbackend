# Product Requirements Document (PRD): Backend Infrastructure for Real-Time Async Service Platform

## 1. Document Overview

### 1.1 Purpose
This PRD defines the requirements for a high-performance backend infrastructure that enables clients (web apps, CLIs, or any WebSocket-capable application) to invoke long-running services/methods and receive results asynchronously via real-time notifications. The system eliminates polling, prioritizes speed in both execution and notification delivery, and supports bidirectional communication.

The backend acts as an RPC-like gateway with job queuing for async tasks, ensuring reliability, scalability, and simplicity.

### 1.2 Scope
- Core backend services: WebSocket gateway, job queuing, worker processing, and result notification.
- Client-agnostic design: Any client that supports native WebSockets.
- Out of scope: Specific business logic of services/methods (to be implemented separately), frontend UI, authentication providers (though auth integration points are defined).

### 1.3 Key Goals
- Zero polling for results — server pushes notifications instantly when tasks complete.
- High speed: Low-latency execution and delivery, leveraging modern runtimes.
- Reliability: Handle disconnects, retries, and failures gracefully.
- Scalability: Support horizontal scaling of gateway and workers.
- Simplicity: Use raw WebSockets without additional abstraction layers.

## 2. High-Level Architecture

```
Clients (Web/CLI/Mobile)
       │
       ▼ (Raw WebSockets over WSS)
WebSocket Gateway (Bun + Fastify + @fastify/websocket)
       │
       ▼ (Enqueue jobs)
BullMQ Queue (backed by Redis with ioredis client)
       │
       ▼ (Workers consume jobs)
Worker Processes (Bun runtime, separate instances)
       │
       ▼ (On completion: Publish results)
Redis Pub/Sub
       │
       ▼ (Gateway subscribes and forwards to specific connections)
WebSocket Gateway → Push results to relevant clients
```

- **Gateway**: Handles client connections, authentication, job submission, and result pushing. Maintains in-memory mapping of jobId → WebSocket connections for targeted notifications.
- **Queue/Workers**: Decouples long-running tasks for scalability and persistence.
- **Redis**: Single source for queue storage and pub/sub notifications.

## 3. Tech Stack

| Component              | Technology                          | Rationale                                                                 |
|-----------------------|-------------------------------------|---------------------------------------------------------------------------|
| Runtime               | Bun (primary)                      | High performance, fast startup/I/O, full Node.js compatibility.           |
| Web Framework         | Fastify                            | Lightweight, high-throughput, excellent plugin ecosystem.                 |
| Real-Time Layer       | Raw WebSockets via @fastify/websocket | Simple, low-overhead, native bidirectional communication.                 |
| Job Queue             | BullMQ                             | Reliable, feature-rich (retries, priorities, delays), Redis-backed.       |
| Redis Client          | ioredis (BullMQ default)           | Robust, supports clustering/TLS, proven in production.                    |
| Redis Backend         | Docker Redis (official redis image, if available) or self-hosted | Containerized for easy local/dev setup and full control; fallback to managed if needed. |
| Security/Transport    | WSS (TLS)                          | Mandatory secure WebSockets.                                              |
| Message Protocol      | JSON-RPC 2.0 (or custom JSON schema with Zod validation) | Structured, versionable RPC over sockets.                                 |
| Monitoring/Logging    | Bun built-in (console logging, runtime metrics) | Simple native capabilities for development and basic production use.      |
| Deployment            | To be decided                      | Flexible based on final environment (e.g., Docker/Kubernetes, cloud).     |

## 4. Functional Requirements

### 4.1 Client Connection & Authentication
- Clients connect via native WebSocket over WSS.
- Authentication: JWT (or similar token) passed in handshake URL query (e.g., `?token=jwt`) or subprotocol. Gateway validates on upgrade request.
- Manual handling of pings/pongs for heartbeats (to detect disconnects).

### 4.2 Service Invocation (RPC)
- Clients send structured JSON messages after connection.
- Request format (JSON-RPC inspired):
  ```json
  {
    "jsonrpc": "2.0",
    "method": "serviceName.methodName",
    "params": { /* method arguments */ },
    "id": "unique-request-id",
    "jobId": "optional-preferred-jobId"
  }
  ```
- Gateway:
  - Validates request and connection auth.
  - Generates jobId if not provided.
  - Enqueues job via BullMQ.
  - Associates the WebSocket connection with the jobId (in-memory Map<jobId, Set<WebSocket>> for multi-client cases).
  - Sends immediate acknowledgment message if requested.

### 4.3 Async Processing & Notifications
- Workers process jobs asynchronously (long-running tasks).
- On completion (success/failure/progress):
  - Worker publishes result to Redis channel (e.g., `job:{jobId}`).
  - Gateway (subscribed to channels) sends message to all associated connections for that jobId:
    ```json
    {
      "jsonrpc": "2.0",
      "result": { /* data */ },
      "error": null,  // or error object
      "id": "matching-request-id",
      "jobId": "jobId"
    }
    ```
- Support intermediate progress updates (worker emits partial events to same channel).

### 4.4 Job Management Features (via BullMQ)
- Automatic retries on failure.
- Job priorities, delays, and timeouts.
- Persistent jobs (survive restarts).
- Client reconnection: On reconnect, client must resubmit or include jobId to re-associate (state not persisted per-connection).

### 4.5 Error Handling
- Graceful handling of disconnects (clean up mappings), timeouts, invalid messages.
- Standardized error responses (JSON-RPC format).
- Manual reconnection logic on client side.

## 5. Non-Functional Requirements

- **Performance**: Sub-100ms notification latency under normal load; Bun-optimized for high concurrency.
- **Scalability**: Horizontal scaling of gateway instances (use Redis for shared queue/pub/sub; sticky sessions or shared connection map if needed).
- **Reliability**: 99.9% uptime target; heartbeats, job persistence.
- **Security**: WSS mandatory, JWT auth, rate limiting per client/IP, message validation.
- **Observability**: Basic Bun console logging and runtime metrics.
- **Cost Efficiency**: Docker Redis for low-cost local/dev; monitor for production.

## 6. Client Usage Examples

Clients use native WebSocket APIs or lightweight libraries.

### 6.1 JavaScript/Web Client Example
```javascript
const socket = new WebSocket("wss://api.example.com/ws?token=jwt-token-here");

socket.onopen = () => {
  console.log("Connected");
  socket.send(JSON.stringify({
    jsonrpc: "2.0",
    method: "data.processHeavyTask",
    params: { input: "large-dataset" },
    id: "req-123"
  }));
};

socket.onmessage = (event) => {
  const response = JSON.parse(event.data);
  if (response.id === "req-123") {
    console.log("Result:", response.result);
  } else if (response.type === "progress") {
    console.log("Progress:", response.percent);
  }
};

socket.onclose = () => {
  console.log("Disconnected - implement reconnect logic");
};
```

### 6.2 Python CLI Client Example (using websocket-client library)
```python
import websocket
import json

def on_open(ws):
    print("Connected")
    ws.send(json.dumps({
        'jsonrpc': '2.0',
        'method': 'analysis.runModel',
        'params': {'data': 'input'},
        'id': 'req-456'
    }))

def on_message(ws, message):
    data = json.loads(message)
    if data.get('id') == 'req-456':
        print("Result:", data.get('result'))

ws = websocket.WebSocketApp("wss://api.example.com/ws?token=jwt-token",
                            on_open=on_open,
                            on_message=on_message)
ws.run_forever(ping_interval=30)  # Manual heartbeat
```

### 6.3 Reconnection & Status Query (Optional Enhancement)
Clients implement exponential backoff reconnect and resend pending jobIds to re-associate.

## 7. Open Questions & Clarifications Needed

To finalize and make this PRD complete, please provide input on the following:

1. **Specific Services/Methods**: What are the initial services or methods the backend needs to support (e.g., data processing, AI inference, file uploads)? This will help define params schemas and worker examples.

2. **Authentication Details**: What auth mechanism/provider (e.g., custom JWT, OAuth, API keys)? User roles/permissions needed?

3. **Expected Load & Scale**: Anticipated concurrent clients, jobs per hour/day? Peak vs. average? This impacts scaling strategy and connection mapping.

4. **Progress Updates**: Do long-running tasks need fine-grained progress reporting (e.g., percentages, stages)? How should workers emit them?

5. **Job Persistence & Querying**: Should clients be able to query past job status/history (e.g., via a separate HTTP endpoint or over socket)?

6. **Deployment Environment**: Preferred cloud/provider, constraints (serverless, etc.)?

7. **Monitoring & Alerts**: Any extensions beyond Bun built-ins?

8. **Additional Features**: Rate limiting per user? File uploads in params? Multi-tenancy? Heartbeat interval/details?

Your answers will help refine the spec and move to implementation planning!