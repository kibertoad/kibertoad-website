---
title: "fauxqs: a Free, Pure-TypeScript SQS/SNS/S3 Emulator"
meta_title: "fauxqs - Free TypeScript SQS SNS S3 Emulator"
description: "Introducing fauxqs, a lightweight in-memory AWS emulator for SQS, SNS and S3 written in pure TypeScript. Faster tests, zero Docker dependency, and full API compatibility."
date: 2026-02-19T00:00:00Z
image: ""
categories: ["TypeScript", "Testing"]
author: "kibertoad"
tags: ["aws", "sqs", "sns", "s3", "testing", "typescript", "emulator"]
draft: false
---

If you are running integration tests against AWS services like SQS, SNS or S3, chances are you've been using [LocalStack](https://localstack.cloud/). It has been the go-to solution for local AWS emulation for years, and for good reason. However, LocalStack recently [announced](https://blog.localstack.cloud/the-road-ahead-for-localstack/) that starting March 2026 the free Community image will require account authentication, and the free tier will no longer include CI/CD credits. For many open-source projects and small teams that rely on LocalStack in their CI pipelines, this is a meaningful change.

This was one of the motivations behind building [fauxqs](https://github.com/kibertoad/fauxqs), a free, MIT-licensed, pure-TypeScript emulator for SQS, SNS and S3. It runs as a single in-process server with no Docker, no Java, no binary dependencies. Point your AWS SDK clients at it and run your tests.

## What Does It Cover?

fauxqs emulates the core functionality of three AWS services on a single endpoint:

**SQS** (18 out of 24 API actions):
- Full message lifecycle: send, receive, delete, visibility timeouts, delay queues, long polling
- Batch operations for send, delete and visibility changes
- [Dead letter queues](https://aws.amazon.com/what-is/dead-letter-queue/) (DLQ) with configurable max receive count
- FIFO queues with message group ordering and deduplication
- MD5 checksums that match the AWS algorithm, so SDK validation passes
- *Not supported: IAM permission management (irrelevant since fauxqs doesn't enforce auth) and message move tasks (an operational feature for bulk-replaying DLQ messages)*

**SNS** (20 out of 24 API actions):
- Topic and subscription management with SNS-to-SQS fan-out delivery
- Filter policies with all the standard operators (prefix, suffix, anything-but, numeric ranges, `$or`)
- Raw message delivery, FIFO topics, batch publish
- *Not supported: platform applications, SMS, and non-SQS delivery protocols*

**S3** (20 out of 23 API actions):
- Bucket and object CRUD with multipart uploads
- ListObjectsV2 with prefix filtering and virtual directories
- Cross-bucket copy, user metadata, bulk delete
- Both path-style and virtual-hosted-style URLs
- *Not supported: versioning, lifecycle policies, encryption, ACLs*

**STS** (1 out of 4 API actions):
- `GetCallerIdentity` only. The AWS SDK and various AWS tooling call this to verify credentials before doing anything else. Without it, they fail immediately when pointed at a local emulator. fauxqs returns a mock identity so they can proceed.

## Performance

Beyond API compatibility, performance was a first-class goal for fauxqs. The real-world validation comes from [message-queue-toolkit](https://github.com/kibertoad/message-queue-toolkit), which has a comprehensive SQS test suite covering queues, topics, subscriptions, DLQs, FIFO ordering, filter policies and batch operations. Running the entire suite end-to-end:

- **LocalStack**: 3 minutes 14 seconds
- **fauxqs**: 1 minute 26 seconds

That's roughly a **2x speedup on a real integration test suite**, with zero changes to the test logic — just swapping the endpoint URL. In CI pipelines where these tests run on every push, halving the feedback loop adds up quickly.

Beyond carefully reviewing code for bottlenecks and clear wins, the speedup comes from fauxqs eliminating the Docker overhead and network hop (it runs in-process, so message operations are essentially function calls routed through a local HTTP server), Node.js is faster than Python (which LocalStack is written in), and fauxqs is built on [Fastify](https://fastify.dev/), one of the fastest HTTP frameworks in the Node.js ecosystem.

### SQS Throughput Benchmarks

To isolate and quantify the raw throughput difference more precisely, fauxqs includes a dedicated [benchmark suite](https://github.com/kibertoad/fauxqs/blob/main/benchmarks/BENCHMARKING.md) that measures single-message SQS operations across four deployment modes:

| Setup | Description |
|-------|-------------|
| **fauxqs-library** | In-process via `startFauxqs()`, no network overhead |
| **fauxqs-docker-official** | Pre-built Docker image with dnsmasq DNS server |
| **fauxqs-docker-lite** | Generic `node:24-alpine` container running `npx fauxqs` |
| **localstack** | LocalStack Docker image with `SERVICES=sqs` |

**Publish 5,000 messages:**

| Setup | Mean | p75 | p99 | Std Dev |
|-------|------|-----|-----|---------|
| fauxqs-library | 3.55s | 3.56s | 3.59s | 38.5ms |
| fauxqs-docker-official | 5.79s | 5.81s | 5.90s | 66.9ms |
| fauxqs-docker-lite | 5.83s | 5.87s | 5.91s | 62.2ms |
| localstack | 10.25s | 10.27s | 10.35s | 63.0ms |

**Consume 5,000 messages:**

| Setup | Mean | p75 | p99 | Std Dev |
|-------|------|-----|-----|---------|
| fauxqs-library | 7.66s | 7.69s | 7.74s | 59.9ms |
| fauxqs-docker-official | 12.18s | 12.21s | 12.21s | 25.4ms |
| fauxqs-docker-lite | 12.21s | 12.22s | 12.24s | 19.5ms |
| localstack | 21.30s | 21.32s | 21.47s | 108.3ms |

**Total wall-clock time (full suite):**

| Setup | Total |
|-------|-------|
| fauxqs-library | 90.69s |
| fauxqs-docker-official | 145.49s |
| fauxqs-docker-lite | 158.24s |
| localstack | 256.32s |

The in-process library mode is roughly **2.8x faster** than LocalStack overall. Even the Docker deployments are about **1.6–1.8x faster**. The benchmarks use sequential single-message operations across 5,000 iterations with [tinybench](https://github.com/tinylibs/tinybench), discarding 1 warmup cycle.

## Who Built It

I'm a principal software engineer at [Lokalise](https://lokalise.com/), lead maintainer of [knex.js](https://github.com/knex/knex), and creator of [node-service-template](https://github.com/kibertoad/node-service-template), with about 8 years of open source experience. Prior to building fauxqs, I spent several years working on [message-queue-toolkit](https://github.com/kibertoad/message-queue-toolkit) (MQT) — a framework-style abstraction layer for message queues that provides reusable base classes, validation, error handling, and lifecycle management for building message-driven services. Basically "message queue best practices as a library" (which will be a topic of a separate blog post).

Working on MQT alongside brilliant engineers like Carlos Gamero and Daria Carlotta Maino meant both implementing common messaging patterns and dealing with a lot of AWS edge cases firsthand. That experience informed what fauxqs needed to get right, and MQT's existing integration tests provided a ready-made validation suite.

100% of message-queue-toolkit tests pass with fauxqs. These tests were previously running on LocalStack, and the migration required no changes to the test logic itself.

## Key Differentiators

- **Significantly faster.** In a real-world test suite ([message-queue-toolkit](https://github.com/kibertoad/message-queue-toolkit)), switching from LocalStack to fauxqs cut execution time roughly in half — from 3 minutes 14 seconds to 1 minute 26 seconds — with no changes to the test logic. Isolated throughput benchmarks show even larger gains, up to 2.8x for in-process usage. See the [Performance](#performance) section for the full breakdown.
- **Message spy system.** A built-in testing primitive for asserting on asynchronous event flows. See the dedicated section below.
- **No Docker required for tests.** fauxqs is a Fastify app that can run directly inside your test process. `npx fauxqs` starts it as a standalone server, but you can also start it programmatically with `startFauxqs()` and avoid spawning a separate process entirely. This simplifies CI setup considerably and eliminates an entire class of "works on my machine" issues related to Docker networking, volume mounts and resource limits. For local development, if you need S3 (see the virtual-hosted-style DNS section below), you'll likely prefer running fauxqs via Docker with the wildcard DNS solution baked in.
- **Pure TypeScript.** No JVM, no Python, no native binaries. If you have Node.js, you can run it. This also means you can embed it directly in your test setup with `startFauxqs()` and get programmatic access to create queues, topics and buckets before tests run.
- **Queue inspection.** You can non-destructively peek at all messages in a queue (ready, in-flight, and delayed) without consuming them or affecting visibility timeouts. Available both as a programmatic API (`server.inspectQueue("my-queue")`) and via HTTP endpoints (`GET /_fauxqs/queues/my-queue`). Useful for debugging test failures when you need to see what's actually sitting in a queue.
- **Init config.** You can pass a JSON file to pre-create queues, topics, subscriptions, and buckets on startup. This is useful for both local development and CI environments.
- **Completely open and free, forever.** MIT-licensed with no plans to monetize any part of it, ever.

## Message Spy: Making Async Testing Feel Like REST Testing

When you test a REST endpoint, the flow is straightforward: send a request, get a response, assert on the response. The whole thing is synchronous from the test's perspective.

Testing event-driven systems is a different story. You publish a message to a queue, and then you need to somehow verify that it actually got delivered, or that a consumer picked it up, or that a poison message ended up in the DLQ. None of these produce a return value you can assert on directly.

The typical workarounds are not great. You can add `setTimeout` delays and hope the async operation completes in time. You can poll in a loop with retries. You can wire up custom event listeners. All of these add noise to your tests and make them either flaky (too short a wait) or slow (too generous a wait).

fauxqs includes an optional message spy system that addresses this. When enabled, it tracks every event flowing through SQS, SNS and S3, and lets you `await` specific events with predicate-based matching.

Here's what testing a publish-and-consume flow looks like:

```typescript
const server = await startFauxqs({ port: 4566, logger: false, messageSpies: true });

// Publish a message
await sqsClient.send(new SendMessageCommand({
  QueueUrl: queueUrl,
  MessageBody: JSON.stringify({ orderId: "order-123" }),
}));

// Assert that the message was published — resolves immediately if already in buffer
const published = await server.spy.waitForMessage(
  (m) => m.service === "sqs" && m.queueName === "orders" && m.body.includes("order-123"),
  "published",
);
expect(published).toBeDefined();
```

The `waitForMessage` call checks the internal buffer first. If a matching event is already there (because the operation completed before the assertion ran), it resolves immediately. If not, it returns a Promise that resolves as soon as a matching event arrives. All `waitFor` calls accept an optional timeout in milliseconds, so your tests won't hang indefinitely if something goes wrong. No polling, no arbitrary sleeps.

This works across the full message lifecycle. You can wait for a message to be consumed (deleted from the queue):

```typescript
// Wait for the message to be consumed by a worker
const consumed = await server.spy.waitForMessage(
  (m) => m.service === "sqs" && m.queueName === "orders",
  "consumed",
);
```

Or wait for a message to be moved to a dead letter queue after exceeding the receive count:

```typescript
const dlqEvent = await server.spy.waitForMessage(
  (m) => m.service === "sqs" && m.queueName === "orders",
  "dlq",
);
```

S3 operations are tracked too. You can wait for an object upload, download, copy or deletion:

```typescript
const uploaded = await server.spy.waitForMessage(
  (m) => m.service === "s3" && m.bucket === "exports" && m.key === "report.csv",
  "uploaded",
);
```

For simpler cases, you can use a partial object matcher instead of a predicate function:

```typescript
const msg = server.spy.checkForMessage(
  { service: "sqs", queueName: "orders", body: "obj match" },
  "published",
);
```

There is also `waitForMessageWithId` for cases where you have the SQS/SNS message ID from the send response and just want to track that specific message through the system:

```typescript
const sendResult = await sqsClient.send(new SendMessageCommand({ ... }));

// Wait for this specific message to be consumed
const consumed = await server.spy.waitForMessageWithId(sendResult.MessageId!, "consumed");
```

When you need to wait for multiple messages (e.g., a batch publish or fan-out), `waitForMessages` collects a specified number of matches before resolving:

```typescript
const msgs = await server.spy.waitForMessages(
  { service: "sqs", queueName: "orders" },
  { count: 3, status: "published", timeout: 5000 },
);
// msgs.length === 3
```

There is also `expectNoMessage` for negative assertions — verifying that something *didn't* happen. This is useful for testing that a filter policy correctly dropped a message, or that a side effect did not occur:

```typescript
// Assert no message was delivered to the wrong queue (waits 200ms by default)
await server.spy.expectNoMessage({ service: "sqs", queueName: "wrong-queue" });

// Custom window and status filter
await server.spy.expectNoMessage(
  { service: "sqs", queueName: "orders" },
  { status: "dlq", within: 500 },
);
```

The spy has zero overhead when disabled (the default). When enabled, it maintains a fixed-size buffer (100 events by default, configurable) with FIFO eviction. You can call `server.spy.clear()` between tests to reset state.

The overall effect is that testing asynchronous message-driven flows starts to feel a lot more like testing REST endpoints: trigger an action, await the outcome, assert. No sleeps, no polling loops, no flakiness from timing issues.

## S3 and Virtual-Hosted-Style Buckets

You might wonder why an SQS/SNS emulator also includes S3. The reason is practical: in the messaging world, S3 is commonly used for implementing the [claim check pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/StoreInLibrary.html) (also known as payload offloading). When your message payload exceeds the SQS size limit of 1 MiB (raised from 256 KB in August 2025), you store the actual payload in S3 and send only a reference through the queue. [message-queue-toolkit supports this natively](https://github.com/kibertoad/message-queue-toolkit?tab=readme-ov-file#payload-offloading), and testing it requires a working S3 alongside SQS and SNS.

Getting S3 to work with local emulators outside of Docker networking has a well-known wrinkle. By default, the AWS SDK sends S3 requests using virtual-hosted-style URLs, where the bucket name is prepended to the hostname: `my-bucket.s3.localhost:4566`. This means DNS needs to resolve `*.localhost` subdomains to `127.0.0.1`, which doesn't happen automatically on all platforms.

Inside Docker, this is a non-issue because container networking handles it. Outside Docker, you need to deal with it explicitly. If you need S3 for local development (not just tests), you'll likely benefit from running the official [fauxqs Docker image](https://github.com/kibertoad/fauxqs) with the wildcard DNS solution baked in, rather than using the embedded library version.

fauxqs provides four options:

**Option 1: `fauxqs.dev` wildcard DNS (recommended if you need S3 beyond tests, e.g. for local development).** A public DNS entry resolves `*.localhost.fauxqs.dev` to `127.0.0.1`, so virtual-hosted-style S3 requests work without `/etc/hosts` changes, custom request handlers, or `forcePathStyle`. This replicates the creative approach [pioneered by LocalStack](https://hashnode.localstack.cloud/efficient-localstack-s3-endpoint-configuration) with their `localhost.localstack.cloud` domain. If you are using the official fauxqs Docker image, this is already the default and you don't need to configure anything. For custom setups, just point your S3 client at it:

```typescript
const s3 = new S3Client({
  endpoint: "http://s3.localhost.fauxqs.dev:4566",
  region: "us-east-1",
  credentials: { accessKeyId: "test", secretAccessKey: "test" },
});
```

The Docker image also includes a built-in DNS server ([dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html)) that resolves the container hostname and all its subdomains to the container's own IP. This means other containers in your docker-compose setup can use virtual-hosted-style S3 URLs (e.g., `my-bucket.s3.fauxqs:4566`) without `forcePathStyle`, by pointing their DNS at the fauxqs container. This is useful when your application container needs to talk to S3 during local development.

**Option 2: `interceptLocalhostDns()` (recommended for embedded library usage).** This patches Node.js `dns.lookup` so that any hostname ending in `.localhost` resolves to `127.0.0.1`. You call it once in your test setup, and every S3Client in the process works without any per-client configuration. Since it only affects `.localhost` subdomains, it's unlikely to interfere with anything else in your tests:

```typescript
import { interceptLocalhostDns } from "fauxqs";

const restore = interceptLocalhostDns();
// ... run tests ...
restore();
```

**Option 3: `createLocalhostHandler()`.** This creates a custom HTTP request handler that resolves all hostnames to `127.0.0.1`. It's scoped to a single S3Client instance, so there are no global side effects. The downside is that you need to pass it to each client individually, which means either conditionally setting `requestHandler` based on test/production context or swapping the S3Client via your DI mechanism:

```typescript
import { S3Client } from "@aws-sdk/client-s3";
import { createLocalhostHandler } from "fauxqs";

const s3 = new S3Client({
  endpoint: "http://s3.localhost:4566",
  region: "us-east-1",
  credentials: { accessKeyId: "test", secretAccessKey: "test" },
  requestHandler: createLocalhostHandler(),
});
```

**Option 4: `forcePathStyle`.** This tells the SDK to use path-style URLs (`http://localhost:4566/my-bucket/key`) instead of virtual-hosted-style, sidestepping the DNS issue entirely. Same caveat as Option 3: you need to conditionally set `forcePathStyle` or replace the client in tests:

```typescript
const s3 = new S3Client({
  endpoint: "http://localhost:4566",
  forcePathStyle: true,
  // ...
});
```

## What It Does Not Do

To be clear: fauxqs is a development and testing tool. It is not intended for running production systems, and you should not use it as a substitute for actual AWS services in any deployed environment.

- All state lives in memory and is lost on restart.
- Authentication is not enforced.
- Non-SQS SNS delivery protocols (HTTP endpoints, Lambda, email) are not supported.
- S3 features like versioning, lifecycle policies and encryption are not implemented.

If you need full AWS parity across dozens of services, LocalStack remains the more comprehensive option.

## Getting Started

Install and run:

```bash
npx fauxqs
```

Or use it programmatically in tests:

```typescript
import { startFauxqs } from "fauxqs";

const server = await startFauxqs({ port: 4566, logger: false, messageSpies: true });

server.setup({
  queues: [{ name: "orders" }, { name: "orders-dlq" }],
  topics: [{ name: "events" }],
  subscriptions: [{ topic: "events", queue: "orders" }],
  buckets: ["uploads"],
});

// ... run your tests against http://localhost:{server.port} ...

await server.stop();
```

Point your AWS SDK clients at the server endpoint and everything works as expected. The project is MIT-licensed and available on [GitHub](https://github.com/kibertoad/fauxqs) and [npm](https://www.npmjs.com/package/fauxqs).

Give it a try, and if you run into any issues or have suggestions, please [open an issue](https://github.com/kibertoad/fauxqs/issues) on GitHub. Contributions are welcome!
