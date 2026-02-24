---
title: "fauxqs: a Free, Pure-TypeScript SQS/SNS/S3 Emulator"
meta_title: "fauxqs - Free TypeScript SQS SNS S3 Emulator"
description: "Introducing fauxqs, a lightweight AWS emulator for SQS, SNS and S3 written in pure TypeScript. In-memory by default with optional SQLite persistence. Runs in-process or via Docker, with full API compatibility."
date: 2026-02-24T00:00:00Z
image: "/images/fauxqs-logo.jpg"
categories: ["TypeScript", "Testing"]
author: "kibertoad"
tags: ["aws", "sqs", "sns", "s3", "testing", "typescript", "emulator"]
draft: false
---

## Introduction

If you are running integration tests against AWS services like SQS, SNS or S3, chances are you've been using [LocalStack](https://localstack.cloud/). It has been the go-to solution for local AWS emulation for years, and for good reason. However, LocalStack recently [announced](https://blog.localstack.cloud/the-road-ahead-for-localstack/) that starting March 2026 the free Community image will require account authentication, and the free tier will no longer include CI/CD credits. For many open-source projects and small teams that rely on LocalStack in their CI pipelines, this is a meaningful change.

This was one of the motivations behind building [fauxqs](https://github.com/kibertoad/fauxqs) ("faux queues"), a free, MIT-licensed, pure-TypeScript emulator for SQS, SNS and S3. It runs as a single in-process server with no Docker, no Java, no binary dependencies. All state is in-memory by default, with optional SQLite-based persistence for local development workflows where you want state to survive restarts. Point your AWS SDK clients at it and run your tests.

## What Does It Cover?

fauxqs emulates the core functionality of three AWS services on a single endpoint. Beyond API compatibility, it aims to replicate the format and structure validation of all inputs that AWS has documented — so if your code would fail against real AWS due to invalid attributes, malformed messages or out-of-range parameters, it will fail the same way against fauxqs.

**SQS:**
- Full message lifecycle: send, receive, delete, visibility timeouts, delay queues, long polling
- Batch operations for send, delete and visibility changes
- [Dead letter queues](https://aws.amazon.com/what-is/dead-letter-queue/) (DLQ) with configurable max receive count
- FIFO queues with message group ordering, per-group locking, deduplication and sequence numbers
- Message attributes with MD5 checksums matching the AWS algorithm
- Queue attribute range validation, message size validation, unicode character validation
- Queue tags
- *Not supported: IAM permission management (irrelevant since fauxqs doesn't enforce auth), dead letter source queue listing, and message move tasks*

**SNS:**
- SNS-to-SQS fan-out delivery to all confirmed subscriptions
- Filter policies with both `MessageAttributes` and `MessageBody` scope, supporting exact match, prefix, suffix, anything-but, numeric ranges, exists, null conditions, and `$or` top-level grouping
- Raw message delivery, FIFO topics, batch publish
- Topic and subscription idempotency with conflict detection
- Subscription attribute validation
- Topic and subscription tags
- *Not supported: permission management, data protection policies, platform applications, SMS, and non-SQS delivery protocols*

**S3:**
- Bucket management: CreateBucket (idempotent, including directory buckets), DeleteBucket, HeadBucket, ListBuckets
- Object operations: PutObject, GetObject, DeleteObject, HeadObject, CopyObject (same-bucket and cross-bucket), RenameObject (directory buckets)
- Multipart uploads with correct multipart ETag calculation, metadata preservation and part overwrite support
- ListObjectsV2 with prefix filtering, delimiter-based virtual directories and continuation tokens
- GetObjectAttributes with selective metadata retrieval
- Stream uploads with AWS chunked transfer encoding support
- Checksums: CRC32, SHA1, SHA256 stored on upload and returned on download, with composite checksums for multipart uploads
- User metadata, bulk delete, keys with slashes
- Both path-style and virtual-hosted-style URLs
- *Not supported: versioning, lifecycle policies, encryption, ACLs, bucket configuration (CORS, replication, logging, website, notifications, policy), object lock, tagging, and public access block*

**STS:**
- `GetCallerIdentity` only. The AWS SDK and various AWS tooling call this to verify credentials before doing anything else. Without it, they fail immediately when pointed at a local emulator. fauxqs returns a mock identity so they can proceed.

## Key Differentiators

- **Significantly faster.** Roughly 1.5x faster on real-world test suites, and up to 2.8x on isolated throughput benchmarks. See the [Performance](#performance) section for the full breakdown.
- **Message spy system.** A built-in testing primitive for asserting on asynchronous event flows. See the dedicated section below.
- **No Docker required for tests.** fauxqs is a Fastify app that can run directly inside your test process. `npx fauxqs` starts it as a standalone server, but you can also start it programmatically with `startFauxqs()` and avoid spawning a separate process entirely. This simplifies CI setup considerably and eliminates an entire class of "works on my machine" issues related to Docker networking, volume mounts and resource limits. For local development, if you need S3 (see the virtual-hosted-style DNS section below), you'll likely prefer running fauxqs via Docker with the wildcard DNS solution baked in.
- **Pure TypeScript.** No JVM, no Python, no native binaries. If you have Node.js, you can run it. This also means you can embed it directly in your test setup with `startFauxqs()` and get programmatic access to create queues, topics and buckets before tests run.
- **Queue inspection.** You can non-destructively peek at all messages in a queue (ready, in-flight, and delayed) without consuming them or affecting visibility timeouts. Available both as a programmatic API (`server.inspectQueue("my-queue")`) and via HTTP endpoints (`GET /_fauxqs/queues/my-queue`). Useful for debugging test failures when you need to see what's actually sitting in a queue.
- **Fast state cleanup.** `server.reset()` clears all messages and S3 objects between tests while preserving resource definitions. No need to tear down and recreate queues, topics and buckets for each test — just reset and go.
- **Init config.** You can pass a JSON file to pre-create queues, topics, subscriptions, and buckets on startup. This is useful for both local development and CI environments.
- **Optional persistence.** By default, all state is in-memory. Pass the `dataDir` option, and fauxqs stores everything in SQLite — queues, messages (including inflight and delayed), topics, subscriptions, buckets, objects, and even FIFO sequence counters. Restart the server and pick up where you left off. See the dedicated section below.
- **Multi-region support.** Region is part of an entity's identity, just like in real AWS. A queue named `my-queue` in `us-east-1` is a completely different entity from `my-queue` in `eu-west-1`. The region is automatically detected from the SDK client's `Authorization` header.
- **Relaxed validation rules.** By default, fauxqs enforces AWS-strict validations. You can selectively relax some of them for convenience during development (e.g., disabling the minimum 5 MiB source size requirement for byte-range `UploadPartCopy`).
- **Completely open and free, forever.** MIT-licensed with no plans to monetize any part of it, ever.

## Performance

Beyond API compatibility, performance was a first-class goal for fauxqs. The real-world validation comes from [message-queue-toolkit](https://github.com/kibertoad/message-queue-toolkit), which has a comprehensive messaging test suite covering queues, topics, subscriptions, DLQs, FIFO ordering, filter policies and batch operations. Running the entire suite end-to-end:

- **LocalStack**: 1 minute 34 seconds
- **fauxqs**: 59 seconds

That's roughly a **1.5x speedup on a real integration test suite**. In CI pipelines where these tests run on every push, shaving a third off the feedback loop adds up quickly.

And this only measures test execution time. There's also the startup overhead: pulling and booting the LocalStack Docker image takes around **30 seconds**, while `startFauxqs()` brings up the Fastify server in about **250 milliseconds**.

Beyond carefully reviewing code for bottlenecks and clear wins, the speedup comes from:

* Eliminating the Docker overhead and network hop — fauxqs runs in-process, so message operations are essentially function calls routed through a local HTTP server
* Node.js being faster than Python, which LocalStack is written in
* fauxqs being built on [Fastify](https://fastify.dev/), one of the fastest HTTP frameworks in the Node.js ecosystem

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

**Total wall-clock time (all iterations across the full suite):**

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

## Beyond Emulation: Features You Won't Find in Other Emulators

fauxqs also includes a few features that go beyond what typical AWS emulators provide.

Since fauxqs runs as a Fastify server you can embed directly in your test process via `startFauxqs()`, it exposes a full programmatic API. You can call `server.setup()` to pre-create queues, topics, subscriptions and buckets before your tests run, or use `server.inspectQueue()` to non-destructively peek at messages without consuming them. Between tests, `server.reset()` clears all messages and S3 objects while preserving your resource definitions — queues, topics, subscriptions and buckets stay intact, so you get a clean slate without the overhead of tearing down and recreating everything. If you do need a full teardown, `server.purgeAll()` removes everything including the resources themselves. For standalone or Docker usage, you can achieve the same with an init config — a JSON file passed at startup that declaratively sets up all your resources.

You can also send messages and publish to topics programmatically, without instantiating SDK clients:

```typescript
// SQS: enqueue a message directly
const { messageId, md5OfBody } = server.sendMessage("my-queue", "hello world");

// With message attributes and delay
server.sendMessage("my-queue", JSON.stringify({ orderId: "123" }), {
  messageAttributes: {
    eventType: { DataType: "String", StringValue: "order.created" },
  },
  delaySeconds: 10,
});

// SNS: publish to a topic (fans out to all SQS subscriptions)
const { messageId: snsMessageId } = server.publish("my-topic", "event payload");
```

Both methods validate inputs (message body, size limits, FIFO deduplication) and emit spy events automatically, so they behave identically to SDK-sent messages from the emulator's perspective.

But the feature most worth exploring in depth is the message spy system.

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

The recommended approach is a **hybrid model**: use fauxqs as an embedded library in your test suites, and use the Docker image for local development. For **tests**, the embedded library mode avoids the DNS issue entirely — you can use `interceptLocalhostDns()` or `createLocalhostHandler()` to handle virtual-hosted-style resolution within Node.js, and you get the full benefit of the programmatic API, message spies, `reset()` between tests, and no Docker overhead. For **local development**, run the official [fauxqs Docker image](https://github.com/kibertoad/fauxqs) which has wildcard DNS baked in via dnsmasq — other containers in your docker-compose setup can use virtual-hosted-style S3 URLs without any per-client configuration.

fauxqs provides four options:

**Option 1: `fauxqs.dev` wildcard DNS (recommended if you need S3 beyond tests, e.g. for local development; less ideal for tests, as Docker adds latency and you lose the programmatic management API).** A public DNS entry resolves `*.localhost.fauxqs.dev` to `127.0.0.1`, so virtual-hosted-style S3 requests work without `/etc/hosts` changes, custom request handlers, or `forcePathStyle`. This replicates the creative approach [pioneered by LocalStack](https://hashnode.localstack.cloud/efficient-localstack-s3-endpoint-configuration) with their `localhost.localstack.cloud` domain. If you are using the official fauxqs Docker image, this is already the default and you don't need to configure anything. For custom setups, just point your S3 client at it:

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

## Persistence

By default, all state is in-memory and lost when the server stops — which is exactly what you want for tests. But for local development, it's convenient to have queues, messages and S3 objects survive restarts. Pass the `dataDir` option, and fauxqs stores everything in a SQLite database:

**CLI:**

```bash
FAUXQS_DATA_DIR=./data npx fauxqs
```

**Docker:**

The Docker image has `FAUXQS_DATA_DIR=/data` preset. Mount a volume and set `FAUXQS_PERSISTENCE=true` to enable it:

```bash
docker run -p 4566:4566 -v fauxqs-data:/data -e FAUXQS_PERSISTENCE=true kibertoad/fauxqs
```

Without `FAUXQS_PERSISTENCE=true`, the server runs in-memory even if a volume is mounted.

**Programmatic:**

```typescript
const server = await startFauxqs({ port: 4566, dataDir: "./data" });
```

All mutations are written through to SQLite immediately — no batching or delayed flush. On restart with the same `dataDir`, the server restores all state including SQS queues with their messages (ready, delayed, and inflight with visibility deadlines), FIFO sequence counters, SNS topics and subscriptions (fan-out works immediately after restart), and S3 buckets with objects and in-progress multipart uploads.

`reset()` and `purgeAll()` also write through to the database, so they work consistently whether you're running in-memory or with persistence enabled.

## What It Does Not Do

To be clear: fauxqs is a development and testing tool. It is not intended for running production systems, and you should not use it as a substitute for actual AWS services in any deployed environment.

- Authentication is not enforced.
- Non-SQS SNS delivery protocols (HTTP endpoints, Lambda, email) are not supported.
- S3 features like versioning, lifecycle policies and encryption are not implemented.

If you need full AWS parity across dozens of services, LocalStack remains the more comprehensive option.

## Migrating from LocalStack

If you're currently using LocalStack and want to try fauxqs, see the [LocalStack migration guide](https://github.com/kibertoad/fauxqs#migrating-from-localstack) in the README. For SNS/SQS-only setups it's a one-line Docker image swap; when S3 is involved there are a few extra steps, and the recommended hybrid path is to use fauxqs as an embedded library for your test suites while keeping Docker Compose for local development environments.

## Examples

The [`examples/`](https://github.com/kibertoad/fauxqs/tree/main/examples) directory in the repository contains runnable TypeScript examples covering fauxqs-specific features beyond standard AWS SDK usage: programmatic API usage, message spy functionality, init configs, queue inspection, Docker standalone and container-to-container communication, and a recommended dual-mode testing approach with vitest.

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

The project is MIT-licensed and available on [GitHub](https://github.com/kibertoad/fauxqs) and [npm](https://www.npmjs.com/package/fauxqs).

Give it a try, and if you run into any issues or have suggestions, please [open an issue](https://github.com/kibertoad/fauxqs/issues) on GitHub. Contributions are welcome!
