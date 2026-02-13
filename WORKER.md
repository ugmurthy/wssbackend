```typescript
// worker/src/worker.ts
// BullMQ Worker Process (run separately from gateway)
// Processes long-running jobs and publishes results/progress via Redis Pub/Sub

import { Worker, QueueEvents } from "bullmq";
import IORedis from "ioredis";

// Redis connection (shared with gateway – use same .env)
const connection = new IORedis(
  process.env.REDIS_URL || "redis://localhost:6379",
);

// Queue name – must match what gateway uses
const QUEUE_NAME = "async-services";

// Placeholder for your actual service implementations
// Add real methods here (e.g., data processing, AI inference)
const services: Record<string, (params: any) => Promise<any>> = {
  "data.processHeavyTask": async (params: any) => {
    // Simulate long-running work
    for (let i = 1; i <= 10; i++) {
      await new Promise((resolve) => setTimeout(resolve, 1000)); // 1s per step
      // Report progress (optional)
      await connection.publish(
        `job:${params.jobId}:progress`,
        JSON.stringify({
          percent: i * 10,
          message: `Step ${i}/10 complete`,
        }),
      );
    }
    return { result: "Processed data", input: params.input };
  },

  // Add more services here...
  // 'analysis.runModel': async (params) => { ... }
};

// Create Worker
const worker = new Worker(
  QUEUE_NAME,
  async (job) => {
    const { method, params, jobId, requestId } = job.data;

    console.log(`Processing job ${jobId} - method: ${method}`);

    try {
      if (!services[method]) {
        throw new Error(`Unknown method: ${method}`);
      }

      // Execute the service
      const result = await services[method]({ ...params, jobId });

      // Publish success result
      await connection.publish(
        `job:${jobId}`,
        JSON.stringify({
          jsonrpc: "2.0",
          result,
          error: null,
          id: requestId,
          jobId,
        }),
      );

      return result;
    } catch (err: any) {
      console.error(`Job ${jobId} failed:`, err);

      // Publish error result
      await connection.publish(
        `job:${jobId}`,
        JSON.stringify({
          jsonrpc: "2.0",
          result: null,
          error: { code: -32603, message: err.message || "Internal error" },
          id: requestId,
          jobId,
        }),
      );

      // Re-throw to let BullMQ handle retries
      throw err;
    }
  },
  {
    connection,
    // Optional: concurrency, etc.
    concurrency: 5,
  },
);

// Optional: Listen to queue events for logging
const queueEvents = new QueueEvents(QUEUE_NAME, { connection });
queueEvents.on("completed", ({ jobId }) =>
  console.log(`Job ${jobId} completed`),
);
queueEvents.on("failed", ({ jobId, failedReason }) =>
  console.log(`Job ${jobId} failed: ${failedReason}`),
);

// Graceful shutdown
process.on("SIGINT", async () => {
  console.log("Shutting down worker...");
  await worker.close();
  await connection.quit();
  process.exit(0);
});

console.log("Worker started – waiting for jobs...");
```

### How to Use This Skeleton

1. Save as `worker/src/worker.ts`.
2. Add real service implementations inside the `services` object.
3. Job data shape expected from gateway:
   ```ts
   {
     method: 'data.processHeavyTask',
     params: { /* args */ },
     jobId: 'uuid',
     requestId: 'client-request-id'
   }
   ```
4. Progress (optional): Publish to `job:${jobId}:progress` – gateway can subscribe and forward.
5. Run with:
   ```bash
   cd worker
   bun run src/worker.ts
   ```

Scale by running multiple worker instances.
