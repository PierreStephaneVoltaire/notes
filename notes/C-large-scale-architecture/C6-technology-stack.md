# C6 Technology Stack
> Last verified: 2026-10 · Audience: senior/staff DevOps, SRE, AI eng, NetEng, DevSecOps, architects

## TL;DR
- **Concurrency model is the core lens.** Thread or process per connection (Apache prefork/worker, classic Tomcat/Jetty) vs **event loop** (Nginx, Node.js, Redis, Netty) vs **virtual threads** (Java 21+). Know where each one blocks and what limits it: RAM per thread, file descriptors, or CPU.
- **Edge tier:** Nginx (or Envoy/HAProxy) in front as reverse proxy, TLS terminator and micro-cache. App servers sit behind it. A CDN (CloudFront / Front Door) sits in front of all of it.
- **Caching:** Memcached is a simple, multi-threaded, volatile LRU cache with a slab allocator. Redis/Valkey is a data-structure server with a single-threaded command core, optional persistence (RDB/AOF), replication and Cluster (16,384 hash slots). Licensing as of 2026: Redis 8 is tri-licensed RSALv2/SSPLv1/**AGPLv3**. **Valkey** (Linux Foundation, BSD-3) is the fork that AWS and others build on.
- **Messaging:** RabbitMQ is a smart broker with exchange routing, per-message acks and **quorum queues** (Raft). Kafka is a dumb broker with smart consumers: a partitioned, replicated log. It is **KRaft-only since 4.0** (ZooKeeper removed). **Share groups (Queues for Kafka, KIP-932)** have been production-ready since 4.2. Redis Pub/Sub is fire-and-forget. Redis Streams are a durable log with consumer groups.
- **Cloud mapping:** S3 ↔ Blob, CloudFront ↔ Front Door, ElastiCache/MemoryDB ↔ Azure Managed Redis, SQS ↔ Storage Queues/Service Bus, SNS/EventBridge ↔ Event Grid, Kinesis/MSK ↔ Event Hubs (Kafka API).
- **2025–2026 changes interviewers like to probe:**
  - AWS App Runner is closed to new customers. AWS points to ECS Express Mode instead.
  - Azure Cache for Redis retires: Enterprise tiers on 2027-03-31, Basic/Standard/Premium on 2028-09-30. The replacement is Azure Managed Redis.
  - Front Door (classic) retires 2027-03-31.
  - The SQS maximum message size is now 1 MiB.
  - S3 has had strong read-after-write consistency since Dec 2020.
- **Datastores:** pick by access pattern and consistency need. RDBMS first: scale with pooling, cache, replicas, then sharding or distributed SQL. Go to NoSQL for known-pattern scale (DynamoDB/Cosmos DB, Cassandra, MongoDB), and model data query-first.
- **Dynamo vs DynamoDB (say this explicitly):**
  - The **2007 Dynamo paper** is leaderless: consistent hashing with vnodes, (N,R,W) sloppy quorum, hinted handoff, vector clocks, Merkle anti-entropy and gossip.
  - **Managed DynamoDB is leader-per-partition**: 3 replicas in 3 AZs with **Multi-Paxos**, and writes are acked on 2/3 (USENIX ATC 2022).
  - Cassandra keeps the Dynamo ring but uses last-write-wins timestamps instead of vector clocks.
- **Bigtable lineage:** a sorted map keyed by (row, column family:qualifier, timestamp), with tablets, LSM storage (memtable → SSTable → compaction) and Chubby. HBase is the open-source copy. Cloud Bigtable nodes are stateless over Colossus.
- **Search and logs:**
  - Shippers (Fluent Bit) → a Kafka buffer → Logstash → Elasticsearch/OpenSearch.
  - Size shards at 10–50 GB and < 200M docs. The primary shard count is fixed when the index is created.
  - Elasticsearch licensing went Apache 2.0 → SSPL/ELv2 (2021) → AGPLv3 added (2024). OpenSearch is the Apache 2.0 fork.
- **Big data:** HDFS (NameNode in RAM, 128 MB blocks, RF 3) → MapReduce (disk between stages) → Spark (DAG of stages split at shuffles, AQE; 4.x has ANSI on by default) → streaming (event time, watermarks, checkpointed state). Kinesis Data Analytics for SQL is gone (apps deleted from 2026-01-27). Azure MongoDB vCore is now **Azure DocumentDB**.

## C6.1 Web applications
- **How it works:** The classic tiers are **client → (CDN) → L4/L7 LB → web server / reverse proxy → app server → cache / DB / queue**.
  - Static content (HTML, JS, images) is served from the edge or object storage.
  - Dynamic content is rendered by app servers, either SSR or JSON APIs for an SPA.
- **Statelessness** is what lets the web/app tier scale horizontally. Session state goes to Redis, signed cookies or JWT, and files go to object storage. Sticky sessions are a fallback, not the plan.
- **Request-cost anatomy:**
  - DNS, then TCP + TLS (1-RTT in TLS 1.3, 0-RTT with QUIC resumption), then the proxy hop, the app handler and the downstream calls.
  - Tail latency comes from fan-out and blocking I/O.
- **Interview angles:**
  - If asked "how do you scale a web app 10×", say: make it stateless, put a CDN in front for static content and cacheable API responses, autoscale the app tier behind an LB, add a cache tier, move slow work to queues, and scale the DB with replicas and sharding last.
  - Pitfall: storing sessions in local memory or on local disk, which breaks scale-in and blue/green.

## C6.2 Solutions for web applications
| Layer | Typical choices | Concurrency model |
|---|---|---|
| Web server / reverse proxy | **Nginx**, Apache httpd, Caddy, HAProxy, Envoy, Traefik | Event-driven (Apache: MPM-dependent) |
| JVM app servers | Tomcat, **Jetty**, Undertow, Netty (WebFlux), Quarkus/Vert.x | Thread-per-request or event loop; virtual threads on Java 21+ |
| JS runtime | **Node.js**, Deno, Bun | Single-threaded event loop + libuv threadpool |
| Python | Gunicorn (pre-fork WSGI), Uvicorn (ASGI/asyncio) | Pre-fork processes / event loop |
| Go | net/http | Goroutine per connection (M:N scheduler) |
| Managed PaaS | Elastic Beanstalk, ECS Express Mode, App Service, Container Apps | Platform-managed |
- **Trade-off:**
  - Thread/process-per-request is simple and isolates failures, but memory grows linearly with concurrency (~1 MB stack per platform thread by default).
  - An event loop handles C10K+ cheaply, but one blocking call stalls every connection on that loop.
- **Interview angle:** Don't put a heavyweight server in front of a heavyweight server. A typical stack is Nginx/Envoy (TLS, buffering, slow-client protection) → app server (business logic).

## C6.3 Apache web server
- **How it works:**
  - Apache HTTP Server ("httpd", 2.4.x line) is a modular server. Modules include `mod_ssl`, `mod_proxy` (`_http`, `_fcgi`, `_balancer`), `mod_rewrite`, `mod_security`, `mod_http2` and `mod_cache`.
  - Dynamic content runs either in-process (`mod_php`, which needs prefork because of thread safety) or out of process via **`mod_proxy_fcgi` → PHP-FPM** (preferred).
  - Per-directory `.htaccess` lets you delegate config without a restart, but httpd then has to stat every directory in the path on every request. Set `AllowOverride None` for performance.
- **When to use:** shared hosting, legacy `.htaccess`-heavy apps, rich module ecosystem (mod_security WAF, auth modules), and teams with an Apache estate.
- **Interview angles:**
  - "Apache vs Nginx?" Apache gives flexibility, per-directory config and in-process modules. Nginx gives a lower memory footprint per connection with an event-driven design.
  - Apache **event MPM** has largely closed the keep-alive gap.

## C6.4 Apache webserver architecture
- **MPMs (Multi-Processing Modules)** decide how connections map to processes and threads. Only **one MPM can be loaded at a time**, and it can be switched via `LoadModule` when built with `--enable-mpms-shared`.

| MPM | Model | Pros | Cons |
|---|---|---|---|
| **prefork** | One single-threaded child process per connection | Isolation; works with non-thread-safe modules (mod_php) | Highest RAM per connection; poor at keep-alive |
| **worker** | N processes × `ThreadsPerChild` threads; one thread per connection | Far less RAM | An idle keep-alive connection still pins a thread |
| **event** | Like worker, plus a **listener thread** per process using epoll/kqueue | Keep-alive, lingering close and blocked writes are handed to the listener, so workers are only busy while actually processing | Needs thread-safe modules |

- **Default MPM on Unix:** `event` if the OS supports threads and thread-safe polling (all modern OSes do), otherwise worker, otherwise prefork.
- **Event MPM connection math:**
  - A process accepts new connections while `connections < ThreadsPerChild + AsyncRequestWorkerFactor × idle_workers`.
  - `AsyncRequestWorkerFactor` defaults to **2**.
  - Worked example: 4 processes × 10 threads (`MaxRequestWorkers` 40) can hold about 120 connections when all workers are idle.
- **Interview angle:** If asked why Apache "fell over at 256 users", the answer is prefork's default `MaxRequestWorkers` (256), with idle keep-alive connections holding processes. Fix: switch to event MPM + PHP-FPM, and tune `KeepAliveTimeout`.

## C6.5 Apache webserver scalability
- **Vertical:**
  - Size `MaxRequestWorkers` ≤ RAM / per-worker RSS, and set `ServerLimit` and `ThreadsPerChild` to match.
  - Set `MaxConnectionsPerChild` to recycle leaky children.
  - Keep `KeepAliveTimeout` low (2–5 s) on prefork/worker.
- **Offload:**
  - Static files go to a CDN or Nginx.
  - App logic moves out of process (PHP-FPM, app servers behind `mod_proxy_balancer`).
  - TLS can be terminated at the LB.
- **Horizontal:** run stateless httpd nodes behind an L4/L7 LB. Share sessions via an external store, and don't keep app state on local disk.
- **Caching:** `mod_cache` + `mod_cache_disk`/`mod_cache_socache`, though Nginx or Varnish is more common in front.
- **Pitfall:** raising MaxRequestWorkers past available RAM makes the box swap, and throughput collapses. Queueing at the LB is better than thrashing.

## C6.6 Nginx webserver
- **What it is:** an event-driven HTTP server, reverse proxy, L7 load balancer, TLS terminator, cache and TCP/UDP proxy (`stream` module).
  - Open source (nginx.org) and NGINX Plus (F5) adds active health checks, cache purge API, JWT auth, dynamic upstreams and a live API.
  - Forks and relatives: Angie (a fork by ex-NGINX developers) and OpenResty (Nginx + LuaJIT).
- **Use cases:** static file serving (`sendfile`, `tcp_nopush`), reverse proxy to app servers, API gateway (rate limiting with `limit_req`/`limit_conn`), HTTP/2 and HTTP/3 (QUIC) termination, and micro-caching.
- **Kubernetes note:** the community **ingress-nginx** controller was announced for retirement, with best-effort maintenance ending March 2026. Migrate to the Gateway API with Envoy-based or other controllers (verify the current status). This is separate from F5's NGINX Ingress Controller.
- **Interview angle:** Nginx is not a "faster Apache". It wins on concurrency per MB, because 10k idle keep-alive connections cost a few MB of RAM rather than 10k threads.

## C6.7 Nginx architecture
- **Master process:**
  - Runs as root. It reads and validates config, binds ports, and spawns and manages workers.
  - `nginx -s reload` (SIGHUP) starts new workers with the new config and gracefully drains the old ones. That gives zero-downtime config changes and binary upgrades (USR2).
- **Worker processes:**
  - Use `worker_processes auto` (one per core), and each worker is single-threaded.
  - Each runs a non-blocking **event loop** over epoll/kqueue, handling thousands of connections as state machines.
  - `worker_connections` defaults to 512. The connection ceiling is roughly `worker_processes × worker_connections`. A proxied request uses 2 connections, and `worker_rlimit_nofile` caps file descriptors.
- **Accept distribution:** `accept_mutex` is off by default since 1.11.3. `listen ... reuseport` (SO_REUSEPORT) gives each worker its own accept queue, which spreads load better.
- **Blocking work:** disk reads that miss the page cache can block a worker. `aio threads` (thread pools) offloads them. There are also cache manager and cache loader helper processes.
- **Interview angles:**
  - "What blocks Nginx?" Slow disk I/O without aio threads, heavy Lua/njs code, and a synchronous resolver at startup.
  - "Why one worker per core?" No context-switch thrash and no locking. Pin workers with `worker_cpu_affinity`.

```mermaid
flowchart LR
  C1["Clients (10k+ conns)"] --> K["Kernel: listen socket(s), SO_REUSEPORT"]
  K --> W1["Worker 1 (epoll loop)"]
  K --> W2["Worker 2 (epoll loop)"]
  K --> W3["Worker N (epoll loop)"]
  M["Master (root): config, bind, signals"] -. spawns/reloads .-> W1
  M -. spawns/reloads .-> W2
  M -. spawns/reloads .-> W3
  W1 --> U["Upstream pool (keepalive)"]
  W2 --> U
  W3 --> U
  W1 --> TP["aio thread pool (disk I/O)"]
  CM["Cache manager / loader"] --> D[("proxy_cache on disk + keys_zone in shm")]
  W2 --> D
```

## C6.8 Nginx as reverse proxy and cache
- **Reverse proxy:**
  - `proxy_pass` to an `upstream` block. Load-balancing methods are round-robin (default), `least_conn`, `ip_hash`, `hash ... consistent` and `random two least_conn`.
  - Passive health checks come from `max_fails`/`fail_timeout`. Active health checks are NGINX Plus only.
  - Turn on `keepalive N` to upstreams, together with `proxy_http_version 1.1` and `proxy_set_header Connection ""`. Since **1.29.7** the `proxy_http_version` default is 1.1 (previously 1.0).
  - `proxy_buffering on` (the default) shields app servers from slow clients. Turn it off for SSE/streaming and LLM token streams.
  - Set `X-Forwarded-For`, `X-Forwarded-Proto` and `Host`. `proxy_next_upstream` defaults to `error timeout`, so be careful retrying non-idempotent POSTs.
- **Cache:**
  - Defined with `proxy_cache_path /data levels=1:2 keys_zone=name:10m max_size=… inactive=10m` (inactive defaults to 10 min). The keys zone lives in shared memory, about **8k keys per MB**, while the bodies live on disk and in the page cache.
  - The default key is `$scheme$proxy_host$request_uri`. `proxy_cache_valid` sets TTLs per status code. Upstream `Cache-Control`/`Expires`/`Set-Cookie` headers are respected.
- **Stampede and resilience controls:**
  - `proxy_cache_lock on`: only one request fills a given key.
  - `proxy_cache_use_stale error timeout updating http_5xx`: serve stale content while the origin is down.
  - `proxy_cache_background_update on`: stale-while-revalidate behaviour.
  - `proxy_cache_revalidate`: conditional GETs to the origin.
  - **Micro-caching** (1–10 s TTL) absorbs spikes on dynamic pages.
- **Pitfalls:** caching personalised responses (missing `Vary`/cookie bypass), purge needing NGINX Plus or a third-party module, and `inactive` evicting content before its TTL expires. See [C1 caching](../C-large-scale-architecture/C1-performance.md#c127-caching-for-performance).

## C6.9 Web containers & Spring framework
- **Servlet container ("web container"):**
  - Tomcat, Jetty and Undertow implement Jakarta Servlet, managing connectors, the request lifecycle, the filter chain and sessions.
  - The jakarta.* namespace replaced javax.* from Jakarta EE 9 (Spring Boot 3+).
- **Classic model: thread-per-request.** Tomcat NIO uses an acceptor plus pollers, then hands the request to a worker thread for its whole duration.
  - Defaults: `maxThreads` **200**, `maxConnections` 8192 (NIO), `acceptCount` 100 (the OS backlog).
  - A blocking JDBC or HTTP call holds a ~1 MB platform thread, so throughput ≈ threads / latency (Little's Law).
- **Spring MVC** (blocking, Servlet) vs **Spring WebFlux** (reactive on Netty event loops, Reactor). WebFlux needs end-to-end non-blocking drivers (R2DBC, reactive clients) to pay off.
- **Virtual threads (JEP 444, GA in Java 21):** `spring.threads.virtual.enabled=true` (Boot 3.2+) runs request handling on virtual threads. `@Async` and the scheduler switch to `SimpleAsyncTaskExecutor`/`SimpleAsyncTaskScheduler`, which ignore pool sizing.
  - This gives blocking-style code with event-loop-like concurrency.
  - **Pinning:** `synchronized` blocks pinned the carrier thread until **JDK 24 (JEP 491)**. Native/JNI frames still pin.
  - The bottleneck moves downstream. Bound concurrency with DB pool size (HikariCP) or semaphores, because "unlimited threads" will DDoS your database.
- **Spring Boot 4 / Framework 7 (Nov 2025):** Jakarta EE 11 / Servlet 6.1 baseline. Undertow support was dropped (unverified — check the Boot 4 release notes).
- **Interview angle:** "Thread pool exhausted at 200 concurrent slow requests" is a Little's Law problem. Fix it with timeouts, bulkheads, and virtual threads or async, not just `maxThreads=2000`.

## C6.10 Jetty & Spring
- **Jetty** (Eclipse) is a lightweight, embeddable server popular in embedded and IoT, Hadoop/Spark UIs, Solr and gRPC/HTTP2 stacks.
  - **Jetty 12** has an async core decoupled from Servlet. It supports EE8, EE9 and EE10 (EE11 in 12.1) environments side by side, plus HTTP/1.1, HTTP/2, HTTP/3 and WebSocket.
- **Threading:**
  - `QueuedThreadPool` (default max 200) plus `ReservedThreadExecutor`.
  - Jetty's "eat what you kill" scheduling runs the selector's task on the same thread to keep CPU caches hot.
  - Jetty 12 can use a virtual-thread executor via `QueuedThreadPool.setVirtualThreadsExecutor`. In Spring Boot, `spring.threads.virtual.enabled` applies to Jetty too.
- **In Spring Boot:** exclude `spring-boot-starter-tomcat` and add `spring-boot-starter-jetty`. Tune with `server.jetty.threads.max/min/idle-timeout` and `server.jetty.max-connections`.
- **Tomcat vs Jetty:** Tomcat is the default and has the broadest ops knowledge. Jetty has a smaller footprint, strong async/HTTP2 and fine-grained embedding. Throughput differences are usually negligible next to app and DB latency.
- **Interview angle:** choose the server by ecosystem and ops familiarity. The real performance levers are timeouts, pool sizes, connection reuse and offloading TLS to the proxy or mesh.

## C6.11 Node.js
- **How it works:**
  - V8 runs JS on **one main thread**. **libuv** provides the event loop and does async network I/O via epoll/kqueue/IOCP (no threads needed).
  - A **threadpool** handles the work that can't be done asynchronously at the OS level: fs, `dns.lookup` (getaddrinfo), crypto (`pbkdf2`, `scrypt`, `randomBytes`), zlib.
  - The threadpool size defaults to **4** via `UV_THREADPOOL_SIZE` (max 1024).
- **Scaling on multiple cores:**
  - The `cluster` module or a process manager (PM2), or one process per container with K8s replicas (preferred).
  - `worker_threads` for CPU-bound JS.
- **Good fit:** I/O-bound APIs, BFF/gateways, WebSockets/real-time, SSR, streaming.
- **Bad fit:** CPU-heavy work on the main thread (JSON parse of large payloads, image processing, sync crypto). It blocks every connection.
- **Versions:** even-numbered majors become LTS. Node 24 has been LTS since Oct 2025, and Node 26 is due for LTS around Oct 2026. Check nodejs.org/en/about/previous-releases.
- **Interview angles:**
  - "DNS lookups slow under load" → `dns.lookup` saturates the 4-thread pool. Raise `UV_THREADPOOL_SIZE`, use `dns.resolve*` (c-ares, no pool), or cache.
  - Monitor **event loop lag/utilisation** (`perf_hooks.monitorEventLoopDelay`).

## C6.12 Node.js event loop
- **Phases, in order, per iteration:**
  1. **timers** (`setTimeout`/`setInterval`)
  2. **pending callbacks** (deferred system I/O errors)
  3. **idle/prepare** (internal)
  4. **poll** (fetch new I/O events, run I/O callbacks; it blocks here when nothing else is pending)
  5. **check** (`setImmediate`)
  6. **close callbacks** (`socket.on('close')`)
- **Microtasks:** `process.nextTick` queue first, then the Promise microtask queue. Both drain completely after each callback, before moving on. Recursive `nextTick` can **starve I/O**.
- **Ordering gotcha:**
  - Inside an I/O callback, `setImmediate` always fires before `setTimeout(…,0)`.
  - From the main module the order is non-deterministic (it depends on process performance).
- **Interview angles:**
  - "p99 spikes but CPU averages 30%" → a long synchronous task blocks the loop. Find it with `--cpu-prof` or clinic.js, then chunk it with `setImmediate` or move it to a worker thread.
  - Backpressure: honour `stream.write()` returning false / `'drain'`, or memory balloons.

```mermaid
flowchart TB
  T["timers: setTimeout/setInterval"] --> P["pending callbacks"]
  P --> I["idle, prepare (internal)"]
  I --> PO["poll: wait for I/O (epoll), run I/O callbacks"]
  PO --> CH["check: setImmediate"]
  CH --> CL["close callbacks"]
  CL --> T
  MQ["After EACH callback: drain process.nextTick queue, then Promise microtasks"] -.-> T
  PO <-->|"fs, dns.lookup, crypto, zlib"| TP["libuv threadpool (default 4)"]
```

## C6.13 Cloud solutions for web †
- **AWS:**
  - **Elastic Beanstalk** is opinionated PaaS on EC2 + ASG + ELB. You still see and own the resources. Deploy policies are all-at-once, rolling, rolling with additional batch, immutable, and traffic splitting.
  - **App Runner** is container/source-to-URL. **It is closed to new customers**: existing customers keep it, and no new features are planned. AWS recommends **Amazon ECS Express Mode** (one API call → Fargate service + ALB + autoscaling, no extra charge).
  - **ECS/Fargate**, **EKS**, **Lambda** + API Gateway / Lambda Function URLs, and **Amplify Hosting** for SPA/SSR frontends.
- **Azure:**
  - **App Service** (Web Apps): Windows/Linux, code or container. Deployment slots with swap give blue/green. Plans go from Free/Basic/Standard to Premium v3/v4 and Isolated v2 (App Service Environment v3).
  - **Container Apps**: serverless containers on managed K8s, with KEDA scale-to-zero, Dapr, revisions with traffic split, and workload profiles (dedicated/GPU).
  - Also AKS, **Functions** (Flex Consumption), and **Static Web Apps**.
- **Interview angles:**
  - Pick PaaS (App Service ↔ Beanstalk/ECS Express) for standard web apps.
  - Pick serverless containers (Container Apps ↔ ECS Express/Fargate, or Cloud Run on GCP) for spiky, scale-to-zero microservices.
  - Pick K8s (EKS ↔ AKS) when you need platform-level control or portability.
  - Gotcha: App Service slots share the plan's compute, so a perf test in a staging slot hurts prod.

## C6.14 Cloud storage †
- **S3 storage classes** (all 11 nines durability):

| Class | AZs | Min duration | Min billable size | Access |
|---|---|---|---|---|
| Standard | ≥3 | – | – | ms |
| Intelligent-Tiering | ≥3 | – | (<128 KB not tiered) | ms. Auto-moves to IA after 30 d and Archive Instant after 90 d. Optional Archive/Deep tiers |
| Standard-IA | ≥3 | 30 d | 128 KB | ms + retrieval fee |
| One Zone-IA | 1 | 30 d | 128 KB | ms. Not AZ-loss resilient |
| **Express One Zone** | 1 (you pick it) | – | – | single-digit ms, up to 10× faster, request cost 50% lower. **Directory buckets**, session auth |
| Glacier Instant Retrieval | ≥3 | 90 d | 128 KB | ms |
| Glacier Flexible Retrieval | ≥3 | 90 d | 40 KB overhead | minutes to hours (restore) |
| Glacier Deep Archive | ≥3 | 180 d | 40 KB overhead | hours (restore) |

- **Azure Blob access tiers** (block blobs only):
  - **Hot**; **Cool** (30 d min); **Cold** (90 d min); **Archive** (180 d, offline, rehydrate up to 15 h at standard or high priority).
  - **Smart tier** automatically moves blobs between hot, cool and cold.
  - Archive works only with LRS/GRS/RA-GRS, not ZRS/GZRS.
  - **Premium block blob** (SSD, a separate account type that can't be tiered) is the closest match to S3 Express One Zone in purpose. But Premium is regional (LRS/ZRS) rather than an AZ-pinned directory bucket.
- **Consistency:**
  - S3 has had **strong read-after-write consistency for all PUT/DELETE/LIST since Dec 2020**.
  - S3 supports conditional writes (`If-None-Match`, `If-Match`) for optimistic concurrency.
  - Azure Blob has always been strongly consistent, with ETag/lease concurrency.
- **Redundancy:**
  - S3 classes are multi-AZ by default. CRR/SRR handle replication.
  - Azure needs an explicit choice: LRS, ZRS, GRS/RA-GRS, GZRS/RA-GZRS.
- **Interview angles:**
  - Lifecycle rules plus minimum-duration penalties: moving small objects to IA costs more.
  - Express One Zone for ML training scratch and shuffle data, co-located with compute in the same AZ.
  - Use presigned URLs / SAS for direct client uploads. Use OAC (CloudFront) or Private Endpoint / Front Door Premium Private Link to keep buckets private.

## C6.15 Cloud CDN †
- **Amazon CloudFront:**
  - Global PoPs plus Regional Edge Caches, with optional **Origin Shield** (a single mid-tier that collapses origin fetches).
  - Cache and origin-request policies control behaviour. **OAC** protects S3 origins, and VPC origins allow private ALB/EC2.
  - Edge compute: **CloudFront Functions** (lightweight JS, viewer events) and **Lambda@Edge** (Node/Python, origin events).
  - Integrates with WAF and Shield.
  - Flat-rate pricing plans that bundle WAF/DDoS were introduced in 2025 (unverified details).
- **Azure Front Door Standard/Premium:**
  - A global L7 entry point that combines CDN, anycast acceleration (split TCP), WAF, and origin load balancing with health probes.
  - **Premium** adds managed WAF rule sets, bot protection, and **Private Link to origins**.
  - **Front Door (classic) retires 2027-03-31.** **Azure CDN Standard from Microsoft (classic) retires 2027-09-30**; no new profiles since Aug 2025. Azure CDN from Edgio was shut down in Jan 2025.
- **Key differences:**
  - Front Door is CDN + global LB/WAF in one resource.
  - On AWS, global L7 failover across regions needs CloudFront origin groups or Route 53 / Global Accelerator.
  - Global Accelerator (anycast L4) ↔ Azure cross-region Load Balancer.
- **Interview angles:** cache keys (avoid busting via query strings/cookies), purge/invalidation cost and latency, versioned asset filenames instead of invalidation, `stale-while-revalidate`, and protecting the origin from direct access. See [I3 Acceleration](../I-dns-tls-acceleration-gaps/I3-acceleration.md) and the CDN overlaps D1.14 and H6.12.

## C6.16 Services
- **Meaning:** the stateful "backing services" behind stateless web/app tiers:
  - **cache** (C6.18–21)
  - **message queue / event stream** (C6.22–26)
  - **datastores** (C6.27+)
  - **search / analytics** (C6.41+)
- In 12-factor terms they are attached resources, addressed via config (URL/credentials) and swappable.
- **Design rules:**
  - Each service has its own SLO, capacity model and failure mode.
  - The app must degrade gracefully: cache miss means fall through to the DB, a queue outage means buffer or reject, and search being down means hide the feature.
- **Interview angle:** draw the synchronous request path short (LB → app → cache → DB), and push everything else off-path via queues and streams.

## C6.17 Services solutions
| Need | Self-managed OSS | AWS | Azure |
|---|---|---|---|
| Ephemeral cache | Memcached | ElastiCache (Memcached) | – (Azure Managed Redis) |
| Cache + data structures | Redis / **Valkey** | ElastiCache (Valkey/Redis OSS), MemoryDB | **Azure Managed Redis** |
| Task queue | RabbitMQ, ActiveMQ | SQS, Amazon MQ | Storage Queues, Service Bus |
| Pub/sub fan-out / events | NATS, Redis Pub/Sub | SNS, EventBridge | Event Grid |
| Event log / streaming | **Kafka**, Pulsar, Redpanda | MSK, Kinesis Data Streams | Event Hubs (Kafka API) |
- **Selection heuristics:**
  - Need routing, per-message ack/retry/DLQ or workflow semantics → **queue** (RabbitMQ/SQS/Service Bus).
  - Need replay, ordering per key, multiple independent consumers or high throughput → **log** (Kafka/Kinesis/Event Hubs).
  - Need sub-ms reads → **cache**.
  - Choose a managed service unless there's a hard requirement (feature, cost at very large scale, portability).

## C6.18 Memcached
- **What it is:** a multi-threaded, in-memory, volatile **key → opaque blob** cache.
  - Defaults: port 11211, `-m 64` MB, max item size 1 MB (`-I` raises it), `-t 4` threads.
  - Commands: get/set/add/replace/cas/incr/decr/touch, with text and meta protocols.
- **No persistence, no replication, no server-side clustering.** Clients shard keys with **consistent hashing** (ketama). Losing a node loses only its keys, and they refill from the DB.
- **When to use:** a simple look-aside cache of rendered fragments or serialized objects, scaling vertically with threads on many cores. One big multi-threaded node is very efficient.
- **Interview angle:** "Why Memcached over Redis?" For a pure cache with large values on many cores, Memcached is simpler and multi-threaded, with predictable memory. Pick Redis when you need data structures, persistence, replication or pub/sub.

## C6.19 Memcached Architecture
- **Slab allocator:**
  - Memory is carved into 1 MB **pages** assigned to **slab classes**. Each class holds fixed-size **chunks**, and chunk sizes grow by the factor `-f` (default **1.25**).
  - An item goes in the smallest chunk that fits. This avoids malloc fragmentation but wastes space inside each chunk.
- **Eviction:** **per-slab-class LRU** (segmented HOT/WARM/COLD since 1.5), plus the background LRU crawler for expired items.
  - "Slab calcification" happens when the value-size distribution shifts. `slab_reassign`/`slab_automove` (on by default in modern versions) rebalances pages.
- **Threads:** a listener thread plus N worker threads (libevent), with fine-grained item locks.
  - **Extstore** spills large values to flash/SSD while keys stay in RAM.
- **Distribution:** entirely client-side (consistent hashing). Add **mcrouter** (Meta) for pools, replication and failover routing.
- **Pitfalls:**
  - Thundering herd on a hot key expiry. Use lease tokens (as Meta does), lock-on-miss or early refresh.
  - Mod-N hashing remaps almost every key when a node is added. Use consistent hashing (see B6.2/D1.21).

## C6.20 Redis Cache & its architecture
- **Core:**
  - A **single-threaded command execution** event loop (epoll), so every command is atomic. Lua scripts / Functions and MULTI/EXEC give multi-key atomicity.
  - **I/O threads** (`io-threads`, since 6.0) parallelise socket read/write and parsing. Redis 8 reworked them for big throughput gains.
  - Background threads handle `UNLINK`/lazyfree and fsync.
- **Data structures:** strings, hashes (per-field TTL in 7.4+/Valkey 9), lists, sets, sorted sets, streams, bitmaps, HyperLogLog, geo.
  - Redis 8 folds JSON, time series, probabilistic types, the Query Engine and **vector sets** into core.
- **Eviction:** `maxmemory-policy` options are `noeviction` (the default), `allkeys-lru/lfu`, `volatile-*`, `allkeys-random`. The algorithm is approximate LRU/LFU by sampling.
- **Persistence:**
  - **RDB**: fork + copy-on-write snapshots (`save 60 1000`). Compact and fast to restart, but you lose the minutes since the last snapshot.
  - **AOF**: logs every write. `appendfsync everysec` is the default and loses at most ~1 s. **Multi-part AOF** since 7.0. Hybrid RDB preamble.
  - On restart AOF wins if both are enabled. A large-heap fork can stall for a few ms up to about 1 s, so watch `latest_fork_usec` and THP.
- **HA and scale:**
  - Async replication. **Sentinel** provides failover for non-cluster setups.
  - **Redis Cluster** has **16,384 hash slots** (`CRC16(key) mod 16384`), clients that understand MOVED/ASK redirects, a gossip bus on port +10000, and multi-key ops only within one slot (hash tags `{user:1}`).
  - Async replication means acknowledged writes can be lost on failover. `WAIT` helps but isn't consensus.
- **Licensing (verified):**
  - Mar 2024: Redis 7.4+ moved from BSD to dual RSALv2/SSPLv1.
  - That triggered the **Valkey** fork from 7.2.4 under the Linux Foundation (BSD-3), backed by AWS, Google and Oracle. Valkey 8.0 (Sep 2024), 8.1 (Apr 2025), 9.0 (Oct 2025, atomic slot migration, hash-field expiry), 9.1 (May 2026).
  - May 2025: **Redis 8 added AGPLv3** as a third option (RSALv2/SSPLv1/AGPLv3).
- **Interview angles:**
  - "Redis is single-threaded, so how does it do 1M ops/s?" In-memory work, no locks, pipelining, I/O threads, and sharding across cores and nodes.
  - Big keys and O(N) commands (`KEYS *`, a huge `HGETALL`) block everyone. Use `SCAN`.
  - Hot keys need client-side caching (`CLIENT TRACKING`) or replication of the key.

```mermaid
flowchart LR
  CL["Cluster-aware client (slot map)"] -->|"CRC16(key) mod 16384"| S1["Shard A primary: slots 0-5460"]
  CL --> S2["Shard B primary: slots 5461-10922"]
  CL --> S3["Shard C primary: slots 10923-16383"]
  S1 -->|async repl| R1["Replica A"]
  S2 -->|async repl| R2["Replica B"]
  S3 -->|async repl| R3["Replica C"]
  S1 <-. "cluster bus gossip (port+10000)" .-> S2
  S2 <-. gossip .-> S3
```

## C6.21 Cloud caching solutions †
- **Amazon ElastiCache:**
  - Engines are **Valkey**, Redis OSS (up to 7.1) and Memcached.
  - **Serverless** (Valkey 7.2+, Memcached 1.6.22+, Redis OSS 7.1) autoscales memory and compute. Public endpoints need IAM auth + TLS 1.3 and Valkey 9.0+.
  - **Node-based** clusters support cluster mode on/off, Multi-AZ auto-failover and Global Datastore (cross-region).
  - Valkey is priced lower than Redis OSS (roughly 20% on nodes, 33% on serverless — verify on the pricing page).
  - Node-based Valkey can enable **durability** (a Multi-AZ transactional log).
- **Amazon MemoryDB** (Valkey/Redis OSS-compatible) is a **durable primary database** with a Multi-AZ transaction log. It has strong consistency on the primary and is slower on writes than ElastiCache.
- **Azure Managed Redis (AMR):**
  - Built on Redis Enterprise software and first-party, so there's no Marketplace component.
  - **Clustered by default** (OSS or Enterprise cluster policy; non-clustered up to 25 GB). **Zone redundant by default**, Entra ID auth.
  - Includes modules (JSON, Search/vector, TimeSeries, Bloom) and **active geo-replication** (multi-region active-active via CRDTs).
  - Tiers: Memory Optimized, Balanced, Compute Optimized, and Flash Optimized.
- **Azure Cache for Redis retirement (verified):**
  - **Enterprise/Enterprise Flash** retire **2027-03-31** and are disabled from 2027-04-01.
  - **Basic/Standard/Premium** retire **2028-09-30** and are disabled from 2028-10-01.
  - Migrate to AMR. Clients must handle cluster mode and cross-slot commands.
- **Interview angles:**
  - ElastiCache ↔ AMR is the equivalence.
  - MemoryDB ↔ AMR with persistence + active geo-replication is the closest match for "Redis as primary DB" (or use Cosmos DB).
  - Cache-aside, write-through and TTL jitter are covered in C1.26–C1.30 and D1.15.

## C6.22 RabbitMQ
- **What it is:**
  - A general-purpose message broker. It speaks native **AMQP 0-9-1**; **AMQP 1.0** has been native since 4.0; MQTT and STOMP come via plugins; streams have their own protocol.
  - The "smart broker / dumb consumer" model handles routing, per-message acks, TTL, DLX, priorities and delayed retry.
- **Current line:** **4.x** (docs at 4.3).
  - 4.0 **removed classic mirrored queues**. Classic queues are now non-replicated, and the CQv1 storage format was removed.
  - Use **quorum queues** or **streams** for HA.
- **Fit:**
  - Task/work queues, RPC, complex routing (topic/headers), per-message retry/DLQ.
  - Moderate throughput, roughly tens of thousands of msgs/s per queue (unverified ballpark).
- **Interview angle:** "RabbitMQ vs Kafka?"
  - RabbitMQ: messages are deleted on ack, flexible routing, consumers are pushed messages with prefetch, and it works per message.
  - Kafka: messages are retained, can be replayed, are ordered per partition, and are consumed by pulling offsets. It works per log.

## C6.23 RabbitMQ architecture
- **Topology:** producer → **exchange** → (bindings with routing key/pattern) → **queue** → consumer. Exchange types:
  - **direct** (exact key)
  - **fanout** (broadcast)
  - **topic** (`*` matches one word, `#` matches zero or more)
  - **headers**
  - default (nameless) direct exchange
  - plus alternate exchanges and **DLX** (dead-letter exchange)
- **Reliability chain:** publisher confirms + `mandatory` flag → durable queue + persistent messages → manual consumer **acks** + **prefetch** (`basic.qos`) → DLX for rejects/TTL.
- **Queue types:**
  - **Classic**: single node, non-replicated in 4.x.
  - **Quorum**: Raft-replicated, default 3 replicas (an odd number survives (n-1)/2 failures). Always durable. **Delivery limit defaults to 20** (poison-message protection). No exclusive/non-durable queues and no global QoS.
  - **Streams** (3.9+): an append-only replicated log with non-destructive reads by offset, large fan-out and replay. **Super streams** are partitioned streams.
- **Cluster:**
  - Nodes share metadata, which is stored in Mnesia or **Khepri** (Raft-based). Khepri is the default in recent 4.x (unverified on which version).
  - Queues have a leader on one node. Use odd cluster sizes (3/5). Partition handling matters (`pause_minority`).
  - Federation and Shovel handle WAN replication.
- **Pitfalls:**
  - Long queues (RabbitMQ is happiest when queues are near-empty).
  - Unbounded prefetch.
  - One huge queue as a bottleneck. Shard with consistent-hash exchanges or super streams.
  - Memory/disk alarms block publishers.

## C6.24 Kafka architecture
- **Core model:**
  - A topic is split into **partitions**. Each partition is an ordered, immutable, append-only log made of segment files, where every record has an offset.
  - Ordering is guaranteed **per partition** only, and the key hash picks the partition.
  - Retention is by time or size, or by **log compaction** (keeps the latest value per key).
  - **Tiered storage** (KIP-405, GA in 3.9) offloads old segments to object storage.
- **Replication:**
  - Each partition has a leader and followers. The **ISR** (in-sync replicas) are the followers within `replica.lag.time.max.ms`.
  - With `acks=all` + `min.insync.replicas=2` + RF=3, an acked write survives the loss of one broker.
  - `unclean.leader.election.enable=false` (the default) prefers unavailability over data loss.
  - Idempotent producers are the default (since 3.0). Transactions give exactly-once read-process-write.
- **Metadata: KRaft.**
  - A Raft quorum of **controllers** (3 or 5, dedicated or combined mode) stores metadata in the `__cluster_metadata` log.
  - **ZooKeeper mode was removed in 4.0** (Mar 2025). To migrate from ZK, go through 3.9 (the bridge release) first, then upgrade to 4.x.
  - 4.0 also needs Java 17 for brokers and Java 11+ for clients.
- **Consumption:**
  - **Consumer groups**: each partition goes to at most one consumer in the group, so parallelism ≤ partition count. Offsets are committed to `__consumer_offsets`.
  - The **next-gen rebalance protocol (KIP-848)** has been GA since 4.0 (`group.protocol=consumer`). The broker-side coordinator does incremental assignment, with no stop-the-world rebalances.
  - **Share groups / Queues for Kafka (KIP-932):** preview in 4.1, **production-ready in 4.2 (Feb 2026)**. Multiple consumers cooperatively read the *same* partition with per-record acquisition locks, acks/release/reject, and delivery counts. That gives queue semantics, with no ordering, scaling beyond the partition count.
- **Versions:** 4.3.1 (Jun 2026) is the latest feature line, and 4.2.2 shipped Sep 2026.
- **Interview angles:**
  - Partition count planning: throughput and consumer parallelism vs file handles, recovery time and rebalance cost. Partitions are hard to reduce, and adding them breaks key→partition mapping.
  - Exactly-once is only end-to-end with idempotent sinks or transactions.
  - Consumer lag is the key SLI. See [M4 Kafka at scale](../M-data-platforms/M4-kafka-at-scale.md).

```mermaid
flowchart LR
  P1["Producer (acks=all, idempotent)"] --> B1
  subgraph KC["Kafka cluster (KRaft)"]
    CQ["Controller quorum (Raft): __cluster_metadata"]
    B1["Broker 1: P0 leader, P1 follower"]
    B2["Broker 2: P1 leader, P2 follower"]
    B3["Broker 3: P2 leader, P0 follower"]
    CQ -.-> B1
    CQ -.-> B2
    CQ -.-> B3
    B1 -->|replicate, ISR| B3
    B2 -->|replicate, ISR| B1
    B3 -->|replicate, ISR| B2
  end
  B1 --> CG1["Consumer group A: 1 consumer per partition"]
  B2 --> CG1
  B3 --> SG["Share group: many consumers per partition, per-record acks"]
  B1 --> OS[("Tiered storage: S3 / Blob")]
```

## C6.25 Redis Pub/Sub
- **Pub/Sub:**
  - `PUBLISH`/`SUBSCRIBE`/`PSUBSCRIBE` is **fire-and-forget, at-most-once**. Nothing is stored, offline subscribers miss messages, and there are no acks.
  - Slow subscribers are disconnected when they exceed `client-output-buffer-limit pubsub` (default 32mb hard / 8mb for 60 s soft).
  - In Cluster, classic PUBLISH is broadcast to every node over the bus. **Sharded Pub/Sub** (`SPUBLISH`/`SSUBSCRIBE`, 7.0+) routes by channel slot and scales.
- **Streams** (`XADD`/`XREADGROUP`/`XACK`/`XAUTOCLAIM`):
  - A persisted append-only log with IDs, consumer groups, a **pending entries list (PEL)**, and re-claiming of stuck messages.
  - At-least-once delivery. Trim with `MAXLEN ~`/`MINID`.
  - Streams are a single key, so they live on one shard. Shard across multiple stream keys for scale.
- **When to use:**
  - Pub/Sub: ephemeral signals (cache invalidation fan-out, presence, live dashboards, WebSocket backplanes).
  - Streams: lightweight durable queues and event logs when Redis is already there.
  - Kafka/Event Hubs: big, replayable, retained, multi-team streams.
- **Interview angle:** "Use Redis Pub/Sub for order events?" No. It loses messages on disconnect. Use Streams with consumer groups, or a real broker.

## C6.26 Cloud MQ solutions †
- **AWS:**
  - **SQS Standard**: at-least-once, best-effort ordering, nearly unlimited TPS. Optional **fair queues** use MessageGroupId for multi-tenant noisy-neighbour isolation.
  - **SQS FIFO**: exactly-once processing with 5-min dedup, ordering per MessageGroupId. 300 TPS per partition without batching; **high-throughput mode** goes up to 70k TPS (700k msgs/s batched) in the largest regions.
  - SQS limits:
    - max message **1 MiB** (the Extended Client via S3 handles up to 2 GB)
    - retention 1 min–14 days (default 4 days)
    - visibility timeout default 30 s, max 12 h
    - delay up to 15 min
    - long polling up to 20 s
    - DLQ via redrive policy
  - **SNS**: push pub/sub fan-out to SQS, Lambda, HTTP, SMS and email. Filter policies. FIFO topics. The SNS → SQS fan-out pattern.
  - **EventBridge**: event bus with content-based rules, schema registry, archive/replay, Pipes, Scheduler, SaaS partner sources and cross-account buses.
  - **Kinesis Data Streams**:
    - shards give 1 MB/s or 1k rec/s in and 2 MB/s out
    - on-demand or provisioned capacity
    - retention 24 h default, up to 365 d
    - Enhanced fan-out for dedicated per-consumer throughput
  - **MSK**: managed Apache Kafka, either Provisioned (Standard or **Express brokers**) or **MSK Serverless**. MSK Connect for connectors.
  - **Amazon MQ**: managed ActiveMQ / RabbitMQ for lift-and-shift of JMS/AMQP/MQTT/STOMP apps.
- **Azure:**
  - **Storage Queues**: simple and cheap. 64 KB messages, huge backlog (account-capacity bound), at-least-once, HTTP.
  - **Service Bus**: an enterprise broker.
    - Queues and topics/subscriptions with SQL and correlation filters.
    - **Sessions** (FIFO per session), DLQ, duplicate detection, scheduled messages and transactions.
    - Message size: Basic/Standard **256 KB**; Premium up to **100 MB** over AMQP (default 1 MB), with 1 MB batches.
    - Queue size up to 80 GB.
    - AMQP 1.0 + JMS 2.0 (Premium).
  - **Event Grid**: push eventing for reactive, discrete events. Namespace topics add pull delivery and an **MQTT broker**. CloudEvents schema.
  - **Event Hubs**: a partitioned log with a **Kafka protocol endpoint** (Standard and above; no broker to run).
    - Basic: 32 partitions, 1-day retention, 256 KB events.
    - Standard: 32 partitions, 7 days, 1 MB.
    - Premium: 100 partitions per hub, 90 days.
    - Dedicated: 1,024 partitions, 90 days, 20 MB.
    - Capture to Blob/ADLS (Avro/Parquet).
  - No first-party managed RabbitMQ: use Service Bus (AMQP 1.0), the Marketplace, or self-host on AKS (RabbitMQ Cluster Operator).
- **Interview angles:**
  - "SQS vs Kinesis?" SQS works per message: workers ack and delete, and it autoscales with no shards. Kinesis is an ordered, replayable shard log for streaming analytics with multiple consumers.
  - "Service Bus vs Event Hubs?" Service Bus carries *commands/messages* with transactional semantics. Event Hubs carries *telemetry/event streams* at high volume.
  - Visibility-timeout / lock-duration tuning plus idempotent consumers is the universal at-least-once recipe.

## C6.27 Datastores
- **Categories, and what each is optimised for:**
  - **Relational** (PostgreSQL, MySQL, SQL Server, Oracle): normalised schema, joins, ACID, ad-hoc SQL.
  - **Key-value** (DynamoDB, Redis, Riak): O(1) access by key, horizontal scale, a narrow query model.
  - **Wide-column** (Bigtable, HBase, Cassandra): sparse rows sorted by key, huge write throughput, time series.
  - **Document** (MongoDB, Couchbase, Cosmos DB): JSON/BSON aggregates, a flexible schema, secondary indexes.
  - **Search** (Elasticsearch/OpenSearch): inverted index for full-text, faceting and log analytics.
  - **Graph** (Neo4j, Neptune, Cosmos DB Gremlin), **time-series** (Timestream, InfluxDB, ADX), **vector** (see [K2](../K-ai-infra-llm/K2-embeddings-vector-databases.md)), **object/blob** (S3/Blob, C6.14).
- **Core distinction:** OLTP (many small, latency-bound reads and writes) vs OLAP (scans and aggregates over columnar data). See [M7](../M-data-platforms/M7-data-warehouses.md).
- **Interview angle:** start from the **access patterns, consistency need, scale and latency SLO**, then pick the store. "Polyglot persistence" is fine, but every extra store adds ops cost, consistency gaps (dual writes, so use CDC/outbox) and an extra on-call surface.

## C6.28 Datastore solutions
| Requirement | Default pick | Why |
|---|---|---|
| Transactions, joins, < a few TB | PostgreSQL/MySQL (RDS/Aurora ↔ Azure Database for PostgreSQL/MySQL, Azure SQL) | ACID, mature tooling |
| Relational at global scale | Spanner, CockroachDB, YugabyteDB, **Aurora DSQL** | Distributed SQL, consensus replication |
| Key-value, unbounded scale, single-digit ms | **DynamoDB** ↔ **Cosmos DB for NoSQL** | Partitioned, serverless |
| Write-heavy time series or wide rows | **Cassandra**, Bigtable, HBase | LSM storage, ordered within a partition |
| Flexible JSON aggregates | **MongoDB** (Atlas), Amazon DocumentDB ↔ Azure DocumentDB | Document model, rich queries |
| Full-text, log search | **Elasticsearch/OpenSearch** | Inverted index |
| Analytics over TB–PB | Lakehouse (Delta/Iceberg) + Spark, or a warehouse | Columnar, MPP |
- **Selection checklist:** query patterns (key lookup, range, ad-hoc), consistency (strong, bounded, eventual), multi-region writes, item/row size, hot-key risk, secondary indexes, backup/PITR, cost model (provisioned vs per-request), licence (SSPL/BSL/AGPL vs Apache).
- **Pitfall:** choosing NoSQL "for scale" while the data is relational and fits on one Postgres node with replicas. Most systems never outgrow a well-tuned RDBMS.

## C6.29 RDBMS
- **How it works:**
  - The storage engine keeps **B+tree** indexes and heap or clustered tables in pages (8 KB in Postgres, 16 KB in InnoDB). A **WAL / redo log** gives durability: commit means the log record is flushed to disk.
  - **MVCC** means readers don't block writers (Postgres keeps old tuple versions plus VACUUM; InnoDB uses undo logs).
  - Isolation defaults: Postgres **Read Committed**, MySQL InnoDB **Repeatable Read**, SQL Server Read Committed (RCSI is the Azure SQL Database default).
- **Strengths:** ACID, constraints, joins, a cost-based optimiser, and decades of tooling.
- **Limits:** a single writer node, so write scale is vertical; schema migrations on huge tables; connection-heavy models (Postgres uses a process per connection).
- **Interview angles:** see [B1 ACID](../B-database-engineering/B1-acid.md), [B2 internals](../B-database-engineering/B2-database-internals.md), [B7 concurrency](../B-database-engineering/B7-concurrency-control.md). A common follow-up is "why is Postgres slow at 5,000 connections?" Answer: process per connection, so use PgBouncer (transaction pooling) or RDS Proxy.

## C6.30 RDBMS scalability architecture
- **Ladder, from cheapest to hardest:**
  1. Tune indexes and queries.
  2. Scale vertically.
  3. Add **connection pooling** (PgBouncer, RDS Proxy, Azure's built-in PgBouncer).
  4. Put a **cache** in front (C6.20).
  5. Add **read replicas** (async, so reads lag. Handle read-your-writes by routing to the primary after a write or by checking the LSN).
  6. **Functional partitioning** (a separate DB per service).
  7. **Horizontal sharding** (Vitess, Citus, app-level) or **distributed SQL**.
- **Cloud-native storage split:**
  - **Aurora** keeps 6 copies across 3 AZs, with a **4/6 write quorum and 3/6 read quorum**. The log is the database, and up to 15 replicas share one storage volume.
  - **Aurora Limitless Database** (sharded PostgreSQL) and **Aurora DSQL** (distributed, active-active) are the AWS options.
  - On Azure, **Azure SQL Hyperscale** (page servers + log service, up to 128 TB, named replicas) and **Azure Database for PostgreSQL elastic clusters** (Citus).
- **Sharding costs:** cross-shard joins and transactions, resharding, hot shards, and global uniqueness (Snowflake IDs). See [B5](../B-database-engineering/B5-database-partitioning.md), [B6](../B-database-engineering/B6-database-sharding.md), [B8 replication](../B-database-engineering/B8-database-replication.md), C2.18–C2.20.
- **Interview angle:** "Scale writes 10×." First batch and queue writes, separate hot tables, and partition by tenant. Shard last, and pick the shard key on the dominant access path (tenant_id, user_id).

## C6.31 NoSQL objectives & trade-offs
- **Objectives:** horizontal scale on commodity nodes, high availability across failures, a flexible schema, and predictable latency at scale.
- **The trade-offs you pay:**
  - Weaker or tunable consistency (BASE). Note that **PACELC** is more useful than CAP: even without a partition, you trade latency against consistency.
  - Limited joins and transactions. Most stores offer only single-item or single-partition atomicity. DynamoDB/MongoDB transactions exist but cost more.
  - **Query-first data modelling**: denormalise and duplicate per access pattern (single-table design in DynamoDB, one table per query in Cassandra).
  - Secondary indexes are either local (scatter-gather reads) or global (async, eventually consistent).
- **LSM-tree vs B-tree:** most write-optimised NoSQL stores (Cassandra, HBase, Bigtable, RocksDB) use **LSM trees**: memtable → immutable SSTables → compaction. Writes are sequential and cheap. Reads may touch many SSTables (Bloom filters help). **Write and space amplification** come from compaction. See [B4](../B-database-engineering/B4-btree-vs-bplustree.md).
- **Interview angle:** name the consistency model and the conflict-resolution strategy explicitly (LWW, vector clocks, CRDTs, single leader per partition). "Eventually consistent" alone is not an answer.

## C6.32 Leaderless key-value store (Dynamo model) †
> **Caveat (must say in an interview):** this is the model in the **2007 Dynamo paper** (Amazon's internal shopping-cart store). It inspired Cassandra, Riak and Voldemort. **Managed Amazon DynamoDB is *not* leaderless.** Per the 2022 USENIX ATC paper, each partition is a **replication group of 3 replicas across 3 AZs** that uses **Multi-Paxos** for leader election and consensus. Only the **leader** (which holds a renewable lease) serves writes and strongly consistent reads. A write is acked once **2 of 3** replicas persist the WAL record. Eventually consistent reads can go to any replica. **Log replicas** (WAL-only, Paxos-acceptor-like) are added to restore the write quorum quickly.
- **Paper goals:** "always writeable" (an add-to-cart must never fail), 99.9th-percentile latency SLOs (about 300 ms), incremental scale, symmetric peers with no master.
- **Tunable quorum (N, R, W):** each key is stored on N nodes (its **preference list**). A write succeeds on W acks and a read on R responses. A typical setting is (3, 2, 2). R + W > N gives quorum overlap, though with sloppy quorums that is *not* a linearizability guarantee.
- **Sloppy quorum + hinted handoff:** if a preference-list node is down, the write goes to the next healthy node on the ring with a **hint**. That node delivers the write back when the owner recovers. Availability wins over strict quorum membership.
- **Versioning:** **vector clocks** detect concurrent writes. Conflicting siblings are returned to the client for **semantic reconciliation** (merge cart items, which is why deleted items can reappear). Clocks are truncated past a size threshold.
- **Repair:**
  - **Read repair** fixes stale replicas seen during a read.
  - **Merkle-tree anti-entropy** per key range compares hash trees, so only the differing ranges get transferred.
- **Interview angles:**
  - "Is DynamoDB leaderless?" → **No.** The 2007 Dynamo paper is leaderless; DynamoDB (2012+) uses leader-per-partition Multi-Paxos. **Global tables** are multi-active across regions with **last-writer-wins** (MREC), and **multi-Region strong consistency (MRSC)** is available (GA 2025, unverified exact date).
  - Leaderless vs leader-based: leaderless gives write availability during failures, at the cost of conflicts and read repair. A leader per partition gives simple strong reads, at the cost of a failover window (the new leader waits out the old lease, a couple of seconds per the paper).

## C6.33 Dynamo architecture: consistent hashing and gossip †
> Same caveat as C6.32: this is the paper's design. DynamoDB uses a partition-metadata service and request routers instead of client-visible ring gossip, and adds **global admission control** and adaptive/burst capacity for hot partitions.
- **Consistent hashing:** MD5 of the key → position on a 128-bit ring. Each physical node owns many **virtual nodes (tokens)**, which spreads load, lets heterogeneous hardware take proportional share, and spreads re-replication when a node fails. The paper's final scheme uses **Q equal-sized partitions** with tokens assigned to nodes, which makes bootstrapping and Merkle trees per range simpler. See [B6.2](../B-database-engineering/B6-database-sharding.md) and D1.21.
- **Coordinator:** any node can coordinate. Usually it's the first healthy node in the key's preference list (partition-aware client or via the LB).
- **Membership and failure detection:**
  - **Gossip**: every second each node exchanges membership/token maps with a random peer. Changes converge in O(log n) rounds.
  - **Seed nodes** prevent logical partitions of the ring.
  - Membership changes are explicit admin operations. Transient failures are handled with local, per-request failure detection plus hinted handoff.
- **Cassandra inherited this:** token ring + vnodes (`num_tokens` defaults to 16 since 4.0), gossip, the **phi-accrual failure detector**, snitches for rack/DC awareness, hinted handoff and Merkle repair. **Unlike Dynamo**, Cassandra uses **last-write-wins timestamps**, not vector clocks.
- **Interview angle:** "Adding a node to a ring" → it takes over token ranges from its neighbours, streams those ranges, and only about 1/n of keys move (vs mod-N hashing, which remaps almost everything).

```mermaid
flowchart LR
  C["Client put(cart:42)"] --> CO["Coordinator = first healthy node in preference list"]
  subgraph RING["Consistent-hash ring, N=3, W=2"]
    A["Node A (token range owner)"]
    B["Node B (replica 2)"]
    X["Node C (replica 3, DOWN)"]
    D["Node D (next healthy on ring)"]
  end
  CO --> A
  CO --> B
  CO -. "sloppy quorum: write with hint for C" .-> D
  D -. "hinted handoff when C recovers" .-> X
  A <-. "gossip membership 1/s" .-> B
  A <-. "Merkle-tree anti-entropy per key range" .-> X
  NOTE["Managed DynamoDB today: per partition 3 replicas in 3 AZs, Multi-Paxos leader, 2 of 3 WAL acks"]
```

## C6.34 BigTable wide-column model †
- **Data model (OSDI 2006 paper):** a sparse, distributed, persistent, **sorted map** `(row key, column family:qualifier, timestamp) → uninterpreted bytes`.
  - Rows are sorted **lexicographically by row key**, so row-key design *is* the index and the partitioning.
  - **Column families** are declared up front, few in number, and are the unit of access control and storage (locality groups). Qualifiers are unlimited and dynamic.
  - **Timestamps** keep multiple versions, with GC policies of "keep last N" or "keep newer than T".
  - **Atomicity is per row only.** There are no multi-row transactions (Cloud Bigtable adds single-row read-modify-write and conditional mutations).
- **Row-key design:** avoid monotonically increasing keys (timestamps at the front cause a hot tablet). Use field promotion (`device#reverse_ts`), salting/hashing, and reversed domains (`com.example.www`) for locality.
- **Cloud Bigtable (managed):** HBase-compatible API, **GoogleSQL queries supported**. Single-cluster instances are strongly consistent. Multi-cluster replication is **eventually consistent** by default (read-your-writes or strong via app-profile routing).
- **Cloud mapping:** no exact AWS/Azure twin. The closest are **Keyspaces / DynamoDB** (AWS), **Cosmos DB for NoSQL / for Apache Cassandra** (Azure), or HBase on EMR/HDInsight.

## C6.35 BigTable architecture
- **Components:**
  - A **master** assigns tablets to tablet servers, balances load, and handles schema changes. Clients rarely talk to it.
  - **Tablet servers** serve reads and writes for about 10–1000 tablets each. A tablet is a contiguous row range (~100–200 MB in the paper) and is split when it grows.
  - **Chubby** (a Paxos-based lock service) handles master election, tablet-server liveness (via locks), the schema, and the root of the location hierarchy.
  - **GFS** (now **Colossus**) stores the SSTables and the commit log.
- **Tablet location, a 3 level B+tree-like lookup:** Chubby file → **root tablet** → **METADATA tablets** → user tablet. Clients cache locations.
- **Write/read path:**
  - A write goes to the commit log (shared per server), then the **memtable**.
  - **Minor compaction** flushes the memtable to an immutable **SSTable**. **Merging compaction** combines SSTables. **Major compaction** rewrites everything into one SSTable and drops deletion markers.
  - A read merges the memtable and the SSTables. **Bloom filters** and the block cache cut disk seeks.
- **Cloud Bigtable:** nodes **don't store data**. They hold pointers to tablets on Colossus, so rebalancing and node failure move only metadata (fast), and capacity scales by adding nodes.
- **Interview angle:** compute–storage separation is why Bigtable, Spanner and Aurora scale and recover quickly. Compare HBase (C6.36), where RegionServers hold leases on regions but the data is also on shared HDFS.

## C6.36 HBase
- **What it is:** the open-source Bigtable on Hadoop.
  - HMaster ↔ master, **RegionServer** ↔ tablet server, **region** ↔ tablet.
  - **ZooKeeper** ↔ Chubby, **HDFS** ↔ GFS, **HFile** ↔ SSTable, **MemStore** ↔ memtable, **WAL** ↔ commit log, `hbase:meta` ↔ METADATA.
- **Consistency:** **strongly consistent per row**, because each region is served by exactly one RegionServer. A RegionServer crash means a **failover gap**: ZK session timeout, WAL split/replay, then reassignment, which takes tens of seconds unless you enable region replicas (timeline-consistent reads).
- **Fit:** random read/write over very large sparse tables in a Hadoop estate. **Phoenix** adds a SQL layer and secondary indexes.
- **Pitfalls:** hot regions from sequential keys (pre-split, salt), compaction storms, JVM GC pauses, and ZK/HDFS dependencies that are all ops burden.
- **Managed:** HBase on **EMR** (can store on S3) ↔ **HDInsight HBase** (on 5.1, see the HDInsight note in Cloud mapping). New designs usually pick Bigtable, DynamoDB/Keyspaces or Cosmos DB instead.

## C6.37 Cassandra
- **Architecture:** Dynamo-style distribution (token ring, vnodes, gossip, leaderless, hinted handoff) combined with a Bigtable-style storage engine (commit log → memtable → SSTables + compaction). Any node is a coordinator.
- **Data model (CQL):** `PRIMARY KEY ((partition key), clustering columns)`. The partition key picks the token/replicas. Clustering columns sort rows *within* a partition. Model **one table per query**.
- **Replication:** `NetworkTopologyStrategy` with an RF per DC (typically 3). Rack awareness comes from the snitch.
- **Releases (verified):** **5.0.x** is current GA (5.0.9, Aug 2026). 4.1.x and 4.0.x are still maintained. 6.0 is not yet released per the download page.
- **Pitfalls:**
  - Large partitions (aim for < 100 MB / < 100k rows, unverified rule of thumb). Bucket by time.
  - **Tombstones**: deletes and TTLs leave markers until `gc_grace_seconds` (default **10 days**). Too many tombstones make reads slow or fail. You **must run repair within gc_grace**, or deleted data resurrects.
  - `ALLOW FILTERING` and secondary-index scans on high-cardinality data.
- **Interview angle:** Cassandra suits write-heavy, multi-DC, always-on workloads with known query patterns (IoT, messaging, time series). It is a bad fit for ad-hoc queries, joins and frequent updates to the same row.

## C6.38 Cassandra features
- **Tunable consistency per request:** `ONE`, `QUORUM`, `LOCAL_QUORUM`, `EACH_QUORUM`, `ALL`, `ANY` (writes only). Strong reads need **R + W > RF**, for example LOCAL_QUORUM + LOCAL_QUORUM with RF=3. Multi-DC apps usually use LOCAL_QUORUM so they don't pay WAN latency.
- **Conflict resolution:** **last-write-wins** by cell timestamp, so clock skew matters (use NTP). There are no vector clocks.
- **Lightweight transactions (LWT):** `INSERT ... IF NOT EXISTS` and `UPDATE ... IF col = x` use **Paxos** at `SERIAL`/`LOCAL_SERIAL`. They cost about 4 round trips (Paxos v2 in 4.1 cuts this), so use them sparingly. **Accord** (CEP-15, general multi-partition ACID transactions) is targeted at a future major release (unverified timing).
- **Repair and anti-entropy:** hinted handoff (`max_hint_window` default 3 h), read repair, and **`nodetool repair`** (Merkle trees; incremental or full). Tools like Reaper automate it.
- **Compaction:** STCS (write-heavy), LCS (read-heavy), TWCS (TTL'd time series), and **UCS** (Unified Compaction Strategy, new in 5.0).
- **5.0 headline features (verified):** **Storage-Attached Indexes (SAI)**, which are usable secondary indexes on multiple columns; **trie memtables and trie SSTables**; a **vector type + ANN search** (on SAI); **dynamic data masking**; and **JDK 17**.
- **Interview angle:** "Make Cassandra strongly consistent" → QUORUM reads and writes give read-your-writes for a single key, but not linearizable compare-and-set. For that you need LWT (SERIAL), and even then it's per partition only.

## C6.39 MongoDB
- **Model:** BSON documents (max **16 MB**) in collections, a flexible schema with optional JSON-schema validation, rich queries, the aggregation pipeline, and secondary, compound, multikey, text, geo, TTL and wildcard indexes. **Atlas Search / Vector Search** run on Lucene-based `mongot`.
- **Transactions:** single-document ops are atomic. Multi-document ACID transactions have existed since 4.0 (replica sets) and 4.2 (sharded), with a 60 s default lifetime. Model to avoid them: embed what's read together.
- **Versions:** the release-notes page lists **9.0** as the current stable release, with 8.3, 8.0 and 7.0 as earlier versions (verify dates). Licence is **SSPL** since Oct 2018, which is why AWS/Azure offer *compatible* engines rather than MongoDB itself.
- **Interview angles:**
  - Embed vs reference: embed bounded 1:few data that is read together. Reference unbounded or shared data.
  - The unbounded-array anti-pattern pushes documents toward the 16 MB limit and turns updates into large rewrites.

## C6.40 MongoDB architecture
- **Replica set:**
  - One **primary** plus secondaries (up to 50 members, 7 voting). Raft-like election (protocol version 1). `electionTimeoutMillis` defaults to **10 s**.
  - The **oplog** is an idempotent, capped operation log that secondaries tail.
  - **Write concern** `w:"majority"` has been the default since 5.0, with journaled writes. Read concern can be `local`, `majority`, `linearizable` or `snapshot`. Read preference can be `primary`, `secondaryPreferred` or `nearest`, among others.
  - **Causal consistency** sessions give read-your-writes against secondaries.
- **Sharding:**
  - **mongos** routers, a **config server replica set (CSRS)**, and shards that are each a replica set.
  - **Shard key** can be ranged or hashed. Data is split into chunks (ranges, default 128 MB since 6.0), and the **balancer** migrates them.
  - **`reshardCollection`** (5.0+) changes the shard key online.
  - A query without the shard key is **scatter-gather** to every shard.
- **Storage engine: WiredTiger:**
  - Document-level concurrency (MVCC), compression (snappy by default, zstd optional), and B-tree storage.
  - Internal cache = max(**50% of (RAM − 1 GB)**, 256 MB), with the OS page cache on top.
  - **Checkpoints every 60 s**, plus a journal (WAL) for durability between checkpoints.
- **Interview angle:** the shard key is close to permanent and drives everything. Use high cardinality, low frequency, a non-monotonic key that matches the dominant query (e.g. `{tenantId:1, _id:1}`). A monotonic `_id`/timestamp key makes one hot shard.

## C6.41 Analytics
- **Purpose:** turning operational data, events and logs into insight. Batch (hours), interactive (seconds) and real-time (sub-second to seconds).
- **Architectures:**
  - **Lambda**: a batch layer for correctness plus a speed layer for freshness. Two code paths.
  - **Kappa**: a single streaming path, with reprocessing by replaying the log.
  - **Lakehouse**: open table formats (Delta/Iceberg/Hudi) on object storage with ACID, plus Spark/SQL engines. See [M1](../M-data-platforms/M1-lakehouse-table-formats.md).
- **Two families in this course:** the **log/search analytics** stack (ELK/EFK: shipper → Logstash/Fluentd → Elasticsearch → Kibana) and the **big-data** stack (HDFS → MapReduce → Spark, then streaming).
- **Interview angle:** separate OLTP from analytics. Use CDC (Debezium, DMS, Fabric mirroring) into the lake or warehouse, and don't run heavy analytics on the primary DB.

## C6.42 Analytics solutions
| Need | OSS | AWS | Azure |
|---|---|---|---|
| Log collection/shipping | Fluent Bit, Fluentd, Logstash, Vector, OTel Collector | CloudWatch agent, **Amazon Data Firehose** (renamed from Kinesis Data Firehose) | **Azure Monitor Agent + DCRs**, Logs ingestion API |
| Log search/analytics | Elasticsearch, OpenSearch, Loki | **OpenSearch Service** (managed + Serverless), CloudWatch Logs Insights | **Log Analytics (KQL)**, Azure Data Explorer, Elastic Cloud on Azure (ISV) |
| Batch big data | Hadoop, Spark, Trino | **EMR** (EC2/EKS/Serverless), Glue, Athena | **Azure Databricks**, **Fabric** (Spark), HDInsight, Synapse |
| Stream processing | Flink, Spark Structured Streaming, Kafka Streams | **Managed Service for Apache Flink** | **Stream Analytics**, Fabric Real-Time Intelligence, Databricks |
| Warehouse | ClickHouse, Druid, Pinot | Redshift | Fabric Warehouse, Synapse dedicated SQL |
- **Interview angle:** the logs pipeline needs a **buffer** (Kafka/Kinesis/Event Hubs) between shippers and indexers, so that indexing back-pressure or an outage doesn't drop logs or OOM the agents.

## C6.43 Logstash architecture
- **Pipeline:** **inputs → filters → outputs** in a JVM process.
  - Inputs: beats, kafka, file, http, syslog, jdbc. Filters: grok, dissect, mutate, date, geoip, json, ruby. Outputs: elasticsearch, kafka, s3, among others.
  - Events flow through **pipeline workers**. `pipeline.workers` defaults to the number of CPU cores, `pipeline.batch.size` to 125, and `pipeline.batch.delay` to 50 ms.
- **Queues:** the default is **in-memory** (bounded, lost on crash). A **persistent queue** (`queue.type: persisted`, a disk-backed page file) gives at-least-once delivery and absorbs bursts. A **dead letter queue** captures events rejected by the Elasticsearch output (mapping errors).
- **Multiple pipelines** (`pipelines.yml`) and **pipeline-to-pipeline** communication handle isolation and distributor/collector patterns.
- **Trade-offs:** powerful parsing, but heavy (JVM, hundreds of MB). Keep it off the edge. Run lightweight shippers (Beats, Elastic Agent, Fluent Bit) on hosts and centralise Logstash. Elastic's **ingest pipelines** in Elasticsearch can replace Logstash for simple parsing.
- **Pitfall:** greedy grok patterns cause CPU blowups. Prefer `dissect` for fixed formats and anchor regexes.

## C6.44 Logstash data streaming architecture
- **Reference flow:** Beats/Elastic Agent/Fluent Bit on hosts → **Kafka** (buffer, replay, fan-out) → Logstash consumer group (parse, enrich) → Elasticsearch/OpenSearch (data streams + ILM) → Kibana/Dashboards. Archive to S3/Blob in parallel.
- **Why Kafka in the middle:**
  - It decouples ingest rate from index rate.
  - It survives an Elasticsearch outage for as long as the retention lasts.
  - Several consumers (SIEM, lake, alerting) can read the same stream.
  - Logstash instances scale horizontally as a consumer group, so parallelism is bounded by partitions.
- **Delivery semantics:** end to end it is **at-least-once**. Use a deterministic document `_id` (a hash of the event) for idempotent indexing if duplicates matter.
- **Interview angle:** size by events/s × average event size × retention × (1 + replicas) for Elasticsearch storage, plus Kafka retention for the outage window you need to cover.

## C6.45 Fluentd
- **Fluentd** (Ruby + C, CNCF **graduated**) uses tag-based routing. Sources → filters/parsers → `<match>` outputs, with **buffer** plugins (memory or file, chunked, retry with backoff). It has 1,000+ external plugins and a memory footprint of > 60 MB.
- **Fluent Bit** (C, CNCF graduated as part of the Fluentd project) is about **450 KB**, has 100+ built-in plugins and native **OTLP** in/out. It is the de facto **Kubernetes DaemonSet** log agent and is used by AWS (FireLens, Container Insights) and others.
- **Pattern:** Fluent Bit per node (tail `/var/log/containers`, add K8s metadata) → optional Fluentd/Fluent Bit **aggregator** → Elasticsearch/OpenSearch, Loki, S3, Kafka, CloudWatch or Azure Monitor.
- **EFK vs ELK:** in Kubernetes stacks, Fluentd/Fluent Bit replace Logstash (lighter, cloud-native). The **OpenTelemetry Collector** is converging on the same role for logs, metrics and traces.
- **Pitfalls:** unbounded memory buffers (use filesystem buffering + `mem_buf_limit`), multiline stack traces (use a multiline parser), and back-pressure from a slow output stalling inputs.

## C6.46 Elasticsearch
- **What it is:** a distributed search and analytics engine on **Apache Lucene**, with a JSON REST API, a query DSL, aggregations, ES|QL and **vector search** (dense_vector, HNSW, quantisation). It is near-real-time: new documents become searchable after a **refresh**, every **1 s** by default.
- **Inverted index:** term → postings list (doc IDs, frequencies, positions). Analyzers (tokenizer + filters) decide the terms. **BM25** is the default relevance model. **Doc values** (columnar, on-disk) power sorting and aggregations. `text` fields are analysed and `keyword` fields are exact.
- **Licence history (verified):**
  - Apache 2.0 until **2021**, when Elastic moved to **SSPL + Elastic License 2.0** (7.11).
  - AWS forked 7.10.2 as **OpenSearch** (Apache 2.0), which moved to the Linux Foundation's OpenSearch Software Foundation in 2024 (unverified month).
  - **Aug 2024:** Elastic added **AGPLv3** as a third option, so Elasticsearch is OSI open source again.
  - **OpenSearch 3.x** is current (3.7.0, Jun 2026), roughly one minor every 8 weeks.
- **Interview angles:**
  - It is not a primary database: no transactions, and mapping changes need a reindex. Keep the source of truth elsewhere and index via CDC or a stream.
  - Watch for **mapping explosion** from dynamic fields (`index.mapping.total_fields.limit` defaults to 1000).

## C6.47 Elasticsearch architecture
- **Cluster and node roles:** `master` (dedicated ×3 for quorum-based election, Zen2 since 7.0), `data_hot/warm/cold/frozen`, `ingest`, `ml`, and coordinating-only nodes.
- **Index → shards:**
  - Each **primary shard** is a Lucene index. Since 7.0 the default is **1 primary + 1 replica**.
  - The primary count is fixed at creation (change it only by `_split`/`_shrink`/reindex). Replicas can change at any time.
  - Routing is `shard = hash(_routing or _id) % number_of_primary_shards`.
  - A replica is never on the same node as its primary.
- **Write path:** the coordinating node routes to the primary. The primary writes the in-memory buffer + **translog**, then replicates to the in-sync copies before acking. A **refresh** (1 s) makes a new searchable segment. A **flush** does a Lucene commit and trims the translog. Background **merges** compact segments.
- **Read path:** query-then-fetch. Every shard (primary or replica) returns its top-k doc IDs and scores, then the coordinator merges and fetches the documents.
- **Sizing (Elastic guidance, verified):**
  - **10–50 GB per shard** and **< 200M docs per shard**.
  - At most 1,000 non-frozen shards per node.
  - Master heap of at least 1 GB per 3,000 indices.
  - Too many small shards is the classic cluster killer.
- **Lifecycle:** **data streams** + **ILM** (rollover by size/age → warm → cold → **frozen on searchable snapshots** in S3/Blob → delete).
- **Interview angles:**
  - "Cluster is red" → at least one primary is unassigned. Check `_cluster/allocation/explain`.
  - Yellow means replicas are unassigned (normal on a single node).
  - Split-brain needs a master quorum; that's why you run 3 dedicated masters.

## C6.48 Hadoop HDFS
- **Architecture:**
  - A **NameNode** keeps the whole namespace and block map **in RAM**, roughly 150 bytes per file/block object, which causes the **small-files problem**.
  - **DataNodes** store **128 MB blocks** (the default `dfs.blocksize`), send heartbeats every 3 s, and send block reports.
- **Replication:** the default factor is **3**. The rack-aware policy puts the first replica on the writer's node, and the second and third on two different nodes in **another rack**. Writes are **pipelined** DataNode → DataNode.
- **Hadoop 3 erasure coding** (e.g. RS-6-3) cuts storage overhead from 200% to **50%**, at the cost of CPU and network for cold data.
- **HA:** an active + standby NameNode share edits via **JournalNodes (QJM, quorum of 3+)**, with **ZKFC** + ZooKeeper for automatic failover. **Federation** splits the namespace across multiple NameNodes.
- **Semantics:** write-once, append-only, high throughput and high latency. It is not POSIX, and there are no random writes.
- **Cloud reality:** compute–storage separation replaced HDFS with **object storage**: S3 (via EMRFS/S3A) ↔ **ADLS Gen2** (Blob with a hierarchical namespace, ABFS driver, atomic directory renames). HDFS lingers for on-prem and for EMR/HDInsight scratch space.

## C6.49 Map-Reduce
- **Model:** `map(k1,v1) → list(k2,v2)` → **shuffle and sort** (group by k2, partitioned with `hash(k2) % R`) → `reduce(k2, list(v2)) → output`.
  - A **combiner** does map-side pre-aggregation to cut shuffle bytes.
  - A custom **partitioner** controls key placement and ordering.
- **Runtime:** YARN (ResourceManager + NodeManagers + a per-job ApplicationMaster).
  - **Data locality** schedules maps on the nodes that hold the HDFS blocks.
  - **Speculative execution** handles stragglers.
  - Failed tasks are re-run from the input, because they are deterministic.
- **Costs:** every job materialises intermediate data to **local disk** and writes its output to HDFS. Multi-stage and iterative pipelines (ML, graph) chain many jobs, so there is lots of I/O. That is why Spark, Tez and Flink replaced it.
- **Interview angles:**
  - **Data skew** (one hot key sends a single reducer to the tail). Fix with salting or a two-phase aggregation.
  - The shuffle is the expensive part in *every* distributed engine. Minimise the bytes shuffled.

## C6.50 Apache Spark
- **Architecture:**
  - A **driver** builds the logical plan. **Catalyst** optimises it, and the result is a physical plan made of a **DAG of stages**. Stage boundaries are **shuffles** (wide dependencies). Narrow transformations pipeline within a stage.
  - Each stage runs one task per partition on the **executors** (JVMs with cores and memory). The cluster manager is YARN, Kubernetes or Standalone.
  - **Lazy evaluation:** transformations only build the plan, and actions trigger jobs.
  - **Fault tolerance:** RDD **lineage** recomputes lost partitions. Shuffle files persist on executors or an external shuffle service.
  - **Tungsten** gives off-heap binary rows and whole-stage codegen. **AQE** (on by default since 3.2) coalesces shuffle partitions, switches join strategies and splits skewed partitions at runtime.
- **Versions (verified):**
  - **4.0** (2025): **ANSI SQL mode on by default**, JDK 17 default and Scala 2.13, **Spark Connect** expansion (thin `pyspark-client`), the **VARIANT** type, SQL UDFs and pipe syntax, the Python Data Source API, and State API v2 for streaming.
  - **4.2.0** is the latest (Jul 2026). 3.5.x still gets maintenance.
- **Interview angles:**
  - Broadcast-hash join for small tables (`spark.sql.autoBroadcastJoinThreshold` defaults to 10 MB) vs sort-merge join.
  - `spark.sql.shuffle.partitions` defaults to 200, so tune it or let AQE handle it.
  - Avoid `collect()` on big data. Cache only data that is reused.
  - Small files hurt; use the table format's compaction. Deep dive in [M2 Spark at scale](../M-data-platforms/M2-spark-at-scale.md) and [M3 Databricks](../M-data-platforms/M3-databricks-platform.md).

## C6.51 Stream processing
- **Core concepts:**
  - **Event time vs processing time.**
  - **Windows**: tumbling, sliding/hopping, session.
  - **Watermarks** bound lateness. Late events are dropped or sent to a side output.
  - **Keyed state** lives in RocksDB/state stores and is checkpointed.
  - **Exactly-once**: Flink uses aligned/unaligned checkpoint **barriers** (Chandy–Lamport style) plus transactional or idempotent sinks.
- **Engines:**
  - **Apache Flink**: true streaming, low latency, rich state and timers.
  - **Spark Structured Streaming**: micro-batch by default, plus a continuous/real-time mode. It has the same DataFrame API as batch.
  - **Kafka Streams**: a library with no cluster. State lives in changelog topics.
  - **Managed**: Managed Service for Apache Flink ↔ Azure Stream Analytics / Fabric Real-Time Intelligence. Databricks and Confluent (Flink) run on both clouds.
- **Retired service:** **Kinesis Data Analytics for SQL** was discontinued. No new applications after **2025-10-15**, and existing apps were deleted from **2026-01-27**. Migrate to Managed Service for Apache Flink (or Flink Studio).
- **Back-pressure** propagates upstream (Flink credit-based flow control). Monitor consumer lag, checkpoint duration and size, and watermark skew.
- **Interview angles:**
  - "Exactly-once end to end?" Only with replayable sources (Kafka offsets in the checkpoint) **and** transactional or idempotent sinks. Otherwise it is at-least-once plus dedup.
  - State-size growth: set TTLs on state.
  - Rescaling needs savepoints and stable operator UIDs.
  - Deep dive in [M5 Stream processing](../M-data-platforms/M5-stream-processing.md) and [M4 Kafka](../M-data-platforms/M4-kafka-at-scale.md).

## Diagrams

### Apache MPM models
```mermaid
flowchart TB
  subgraph PF["prefork"]
    PF1["Child proc 1: 1 conn"]
    PF2["Child proc 2: 1 conn"]
    PF3["Child proc N: 1 conn"]
  end
  subgraph WK["worker"]
    WP1["Proc 1: 25 threads, 1 conn per thread incl. idle keep-alive"]
    WP2["Proc 2: 25 threads"]
  end
  subgraph EV["event"]
    EL["Listener thread (epoll): idle keep-alive, lingering close, blocked writes"]
    ET["Worker threads: only while processing a request"]
    EL -->|"readable request"| ET
    ET -->|"response flushed, park socket"| EL
  end
```

### Queue vs log vs pub/sub semantics
```mermaid
sequenceDiagram
  participant P as Producer
  participant Q as "Queue (SQS/RabbitMQ quorum/Service Bus)"
  participant L as "Log (Kafka/Kinesis/Event Hubs)"
  participant C1 as Consumer A
  participant C2 as Consumer B
  P->>Q: send m1
  Q->>C1: deliver m1 (invisible/locked)
  C1->>Q: ack/delete m1 (gone for everyone)
  P->>L: append e1 at offset 42
  C1->>L: fetch from offset 42 (group A)
  C2->>L: fetch from offset 0 (group B, replay)
  Note over L: e1 retained until retention/compaction, independent offsets per group
```

### DynamoDB today: leader-based partition replication (contrast with the Dynamo ring in C6.33)
```mermaid
sequenceDiagram
  participant RR as "Request router"
  participant L as "Leader replica (AZ-a, lease)"
  participant F1 as "Replica (AZ-b)"
  participant F2 as "Replica or log replica (AZ-c)"
  RR->>L: PutItem (partition from metadata)
  L->>L: append WAL record
  L->>F1: replicate WAL (Multi-Paxos)
  L->>F2: replicate WAL
  F1-->>L: persisted
  L-->>RR: ack after 2 of 3 WAL persisted
  RR->>F2: eventually consistent GetItem (any replica)
  RR->>L: strongly consistent GetItem (leader only)
```

### Elasticsearch cluster: shards, replicas, write path
```mermaid
flowchart TB
  CL["Client bulk index"] --> CO["Coordinating node: shard = hash(_id) % primaries"]
  subgraph ES["Cluster (3 dedicated masters elect via quorum)"]
    M["Master nodes x3: cluster state"]
    N1["Data node 1: P0, R1"]
    N2["Data node 2: P1, R2"]
    N3["Data node 3: P2, R0"]
  end
  CO --> N1
  N1 -->|"replicate to in-sync copy"| N3
  N1 -. "buffer + translog, refresh 1s = new Lucene segment" .-> SEG[("Segments, merged in background")]
  M -. "allocates shards, never P and R on same node" .-> N2
  N3 -. "ILM: hot to warm to frozen" .-> SS[("Searchable snapshots on S3 / Blob")]
```

### Spark job: DAG split into stages at shuffles
```mermaid
flowchart LR
  R["read parquet (scan)"] --> F["filter + select (narrow)"]
  F --> X1{{"shuffle: groupBy key"}}
  X1 --> A["aggregate (stage 2)"]
  S["read dim table (small)"] --> B["broadcast"]
  A --> J["broadcast hash join (no shuffle)"]
  B --> J
  J --> W["write Delta/Iceberg (action triggers job)"]
  D["Driver: Catalyst plan, DAG scheduler, AQE"] -.-> X1
```

### Reverse proxy + cache request flow
```mermaid
sequenceDiagram
  participant U as Client
  participant CDN as "CDN (CloudFront/Front Door)"
  participant N as "Nginx (proxy_cache)"
  participant A as "App (Tomcat/Jetty/Node)"
  participant R as "Redis/Valkey"
  participant DB as Database
  U->>CDN: GET /product/1
  CDN-->>U: HIT (edge)
  CDN->>N: MISS -> origin fetch
  N->>A: cache MISS (proxy_cache_lock: 1 filler)
  A->>R: GET product:1
  R-->>A: miss
  A->>DB: SELECT
  DB-->>A: row
  A->>R: SET product:1 EX 300
  A-->>N: 200 Cache-Control max-age=60
  N-->>CDN: 200 (stored, served stale on 5xx)
  CDN-->>U: 200
```

## Cloud mapping: AWS vs Azure
| Capability | AWS | Azure | Role it plays | Key differences | Alternatives |
|---|---|---|---|---|---|
| Web PaaS (code) | Elastic Beanstalk | App Service (Web Apps) | Managed runtime for web apps | Beanstalk exposes the underlying EC2/ASG/ELB. App Service is fully abstracted, with deployment slots + swap | Heroku, Render, GCP App Engine |
| Serverless containers | **ECS Express Mode** / Fargate (App Runner closed to new customers) | **Container Apps** | Container → HTTPS URL with autoscale | ACA has KEDA scale-to-zero, Dapr and revisions. Express Mode provisions an ALB + Fargate in your account | GCP Cloud Run, Knative on K8s |
| Managed K8s | EKS | AKS | Full container orchestration | AKS has a free control-plane tier. EKS charges per cluster-hour | Self-managed K8s, GKE |
| FaaS web backends | Lambda + API Gateway / Function URLs | Functions (Flex Consumption) + API Management | Event-driven HTTP | Cold-start and VNet behaviour differ by plan | Cloudflare Workers |
| Static / SPA hosting | Amplify Hosting, S3 + CloudFront | Static Web Apps, Blob static website + Front Door | Frontend delivery | SWA bundles managed Functions APIs | Cloudflare Pages, Vercel, Netlify |
| Object storage | S3 (Standard, IA, One Zone-IA, Intelligent-Tiering, Glacier IR/Flexible/Deep) | Blob Storage (Hot/Cool/Cold/Archive, smart tier) | Durable blobs, data lake | S3 classes are per object and multi-AZ by default. Azure redundancy (LRS/ZRS/GRS/GZRS) is per account. Archive isn't allowed on ZRS | GCS, Cloudflare R2 (no egress fees), MinIO |
| Low-latency object storage | **S3 Express One Zone** (directory buckets, single AZ) | **Premium block blob** (SSD account) | ML scratch, analytics shuffle | Express is AZ-pinned with session auth. Premium is regional LRS/ZRS and can't be tiered | Local NVMe, FSx for Lustre ↔ Azure Managed Lustre |
| CDN / edge | CloudFront (+ Origin Shield, CloudFront Functions, Lambda@Edge) | **Front Door Standard/Premium** (classic retires 2027-03-31; CDN classic 2027-09-30) | Edge caching, TLS, WAF, acceleration | Front Door bundles global L7 LB + WAF + CDN. AWS uses CloudFront + Route 53/Global Accelerator for multi-region failover. Private origin: CloudFront VPC origins ↔ AFD Premium Private Link | Cloudflare, Akamai, Fastly |
| Reverse proxy / L7 LB (regional) | ALB | Application Gateway (WAF v2) | L7 routing in front of app servers | AppGW is deployed in its own subnet. ALB is ENI-based across AZs | Nginx, Envoy, HAProxy, Gateway API |
| Cache (Memcached) | ElastiCache for Memcached | – (no Memcached service; use AMR) | Volatile look-aside cache | Azure dropped Memcached as a first-party option | Self-host Memcached on VMs/AKS |
| Cache (Redis-compatible) | ElastiCache for **Valkey** / Redis OSS (serverless or node-based, Global Datastore) | **Azure Managed Redis** (Azure Cache for Redis retiring 2027/2028) | Cache, sessions, rate limiting, leaderboards | AMR is Redis Enterprise-based with modules + active-active geo, clustered by default. ElastiCache is Valkey-first and cheaper on Valkey | Redis Cloud, Upstash, Dragonfly, Momento |
| Durable Redis primary DB | **MemoryDB** (Valkey/Redis OSS) | AMR with persistence + geo-replication (closest), or Cosmos DB | In-memory primary DB with durability | MemoryDB commits to a Multi-AZ transaction log before ack. AMR persistence is RDB/AOF to disk | Redis Enterprise, Aerospike |
| Simple queue | **SQS** (Standard/FIFO) | **Storage Queues** | Decoupled work queue | SQS: 1 MiB messages, FIFO, DLQ redrive. Storage Queues: 64 KB, no FIFO/DLQ semantics built in | RabbitMQ, Redis Streams |
| Enterprise broker | SQS FIFO + SNS, or **Amazon MQ** (ActiveMQ/RabbitMQ) | **Service Bus** (queues/topics, sessions, transactions) | Commands, workflows, JMS/AMQP | Service Bus has native topics with filters, sessions, dedup and 100 MB messages (Premium). Amazon MQ is broker-on-instances for protocol compatibility | RabbitMQ, ActiveMQ Artemis, Solace |
| Pub/sub fan-out | **SNS** | Service Bus topics / Event Grid | 1→N delivery | SNS pushes to endpoints. Service Bus subscriptions are pull with durable per-subscriber queues | NATS, Google Pub/Sub |
| Event routing bus | **EventBridge** (rules, archive/replay, Pipes, Scheduler) | **Event Grid** (push, CloudEvents, namespace pull + MQTT) | Reactive integration and SaaS events | EventBridge has richer content filtering + replay. Event Grid has MQTT broker + Azure resource events | Knative Eventing, Kafka Connect |
| Event streaming (log) | **Kinesis Data Streams** | **Event Hubs** | Ordered, partitioned, retained telemetry | Kinesis shards have per-shard limits. Event Hubs uses TU/PU/CU units and speaks the Kafka protocol | Kafka, Redpanda, Pulsar |
| Managed Kafka | **MSK** (Provisioned Standard/Express, Serverless) | **Event Hubs Kafka endpoint** (not real Kafka brokers) or Confluent Cloud on Azure (Marketplace) | Kafka API without ops | MSK is real Apache Kafka (KRaft). Event Hubs implements the protocol, so some admin APIs, compaction limits and Streams features differ | Confluent Cloud, Aiven, Strimzi on K8s |
| MQTT / IoT ingest | IoT Core | Event Grid MQTT broker / IoT Hub | Device messaging | Event Grid MQTT supports MQTT v3.1.1/v5 with routing to Event Hubs | EMQX, HiveMQ |
| Managed relational | RDS, **Aurora** (Limitless, DSQL) | Azure SQL Database (Hyperscale), Azure Database for PostgreSQL/MySQL Flexible Server (elastic clusters) | OLTP | Aurora storage has 6 copies across 3 AZs (4/6 write quorum). Hyperscale uses page servers + a log service, up to 128 TB | Spanner, CockroachDB, YugabyteDB |
| Serverless key-value / document | **DynamoDB** (+ global tables MREC/MRSC) | **Cosmos DB for NoSQL** | Planet-scale KV, single-digit ms | DynamoDB is leader-per-partition Multi-Paxos with 2 consistency choices. Cosmos DB has 5 consistency levels and multi-region writes | ScyllaDB Alternator, Bigtable |
| Cassandra-compatible | **Amazon Keyspaces** (serverless) | **Cosmos DB for Apache Cassandra** (API on the Cosmos engine) / **Azure Managed Instance for Apache Cassandra** (real OSS Cassandra up to 5.0) | Wide-column, CQL | Keyspaces: 3 AZ replicas, writes always LOCAL_QUORUM, reads ONE/LOCAL_ONE/LOCAL_QUORUM only (no QUORUM/ALL/SERIAL levels). Cosmos Cassandra API is wire-compatible but not Cassandra. MI runs real Cassandra in your VNet and supports hybrid rings | DataStax Astra, ScyllaDB, self-managed on K8s (K8ssandra) |
| MongoDB-compatible | **Amazon DocumentDB** | **Azure DocumentDB** (formerly Cosmos DB for MongoDB vCore; built on the MIT-licensed DocumentDB engine on PostgreSQL) / Cosmos DB for MongoDB (RU) | Document store | Neither runs MongoDB server code (SSPL), so check compatibility per operator and feature | **MongoDB Atlas** (on AWS and Azure), FerretDB |
| Wide-column (Bigtable/HBase) | HBase on EMR (S3 storage); Keyspaces/DynamoDB for new builds | HBase on HDInsight; Cosmos DB for new builds | Sorted sparse tables | No first-party Bigtable equivalent on either | Cloud Bigtable (GCP) |
| Search / log analytics | **OpenSearch Service** (managed clusters + Serverless) | **Elastic Cloud on Azure** (Azure Native ISV service), **Azure AI Search** (app/RAG search, not log analytics), Log Analytics/ADX for logs | Full-text, logs, vectors | OpenSearch Service is OpenSearch (or legacy ES ≤ 7.10). Azure has no first-party managed Elasticsearch; AI Search is a different product with its own API | Elastic Cloud, self-hosted ECK/OpenSearch operator |
| Hadoop / Spark platform | **EMR** (on EC2, EKS, Serverless), Glue | **Azure Databricks**, **Microsoft Fabric** (Spark), HDInsight 5.1 (Spark 3.3), Synapse Spark | Batch ETL, ML prep | HDInsight 4.0/5.0 retired 2025-03-31. 5.1 has no announced retirement but ships old Spark. HDInsight on AKS was retired (early 2025, unverified). New Azure designs use Databricks or Fabric | Databricks on AWS, Dataproc |
| Stream processing | **Managed Service for Apache Flink** (KDA for SQL discontinued 2026-01-27) | **Azure Stream Analytics** (SQL, SU-based), Fabric Real-Time Intelligence / Eventstream | Windowed aggregation, CEP | MSF is real Flink (Java/Python/SQL). ASA is proprietary SQL on Trill with exactly-once for selected outputs | Confluent Cloud for Flink, Databricks Structured Streaming |
| Log ingestion pipeline | CloudWatch agent → **CloudWatch Logs** (subscription filters) → **Amazon Data Firehose** → S3/OpenSearch/Splunk | **Azure Monitor Agent + Data Collection Rules** (KQL transforms) / **Logs ingestion API** → Log Analytics | Collect, transform, route logs | DCRs do ingestion-time KQL filtering/masking (cuts cost). Firehose does buffering + Lambda transforms + format conversion | Fluent Bit, OTel Collector, Vector, Cribl |

- **Web hosting:**
  - Beanstalk is IaaS-shaped PaaS: you can SSH in and the resources live in your account.
  - App Service is fully managed. Slot swaps warm up instances before swapping.
  - App Runner closed to new customers in 2026. New designs on AWS should use ECS Express Mode (or Fargate/EKS).
  - Container Apps is the default "serverless containers" answer on Azure.
- **Storage:**
  - S3 makes you choose a class per object, and lifecycle transitions are cheap. Azure tiers are per blob, but redundancy is set per account.
  - Both have early-deletion minimums (30/90/180 d). S3 also has a 128 KB minimum billable size for IA/Glacier IR.
  - Strong consistency on both.
  - S3 Express One Zone is not resilient to AZ loss, so treat it as a cache or scratch tier.
- **CDN:**
  - CloudFront is a pure CDN + edge compute.
  - Front Door is CDN + global anycast L7 LB + WAF. Use it as the single global entry point in Azure designs.
  - Plan the classic-tier migrations (Front Door classic 2027-03-31, CDN classic 2027-09-30).
- **Caching:**
  - AWS is Valkey-first, a result of the 2024 Redis license change.
  - Azure moved to Redis Enterprise (AMR), which is clustered by default. Clients must support cluster mode and avoid cross-slot multi-key ops (or use hash tags).
  - Retirement dates: Enterprise 2027-03-31, Basic/Standard/Premium 2028-09-30.
- **Messaging:**
  - The closest pairs are SQS ↔ Service Bus queues (or Storage Queues for simple, cheap cases), SNS ↔ Service Bus topics / Event Grid, EventBridge ↔ Event Grid, Kinesis ↔ Event Hubs, and MSK ↔ Event Hubs (Kafka API).
  - Event Hubs is Kafka-*compatible*, not Kafka. Verify the feature parity you need (transactions and compaction exist on certain tiers; check the current docs).
  - Azure has no Amazon MQ equivalent.
- **Alternatives:**
  - Cloudflare (CDN, R2, Queues, Workers).
  - Confluent Cloud (multi-cloud Kafka with Flink).
  - Strimzi or the RabbitMQ Cluster Operator on Kubernetes for portability.
  - Self-managed Valkey for licence-clean Redis.
- **Datastores (C6.27–C6.40):**
  - DynamoDB ↔ Cosmos DB for NoSQL is the canonical pair.
  - **Cassandra** has two Azure answers. Use the *API* (Cosmos DB for Apache Cassandra) for serverless/RU economics and global distribution. Use **Managed Instance** when you need real Cassandra behaviour (repair, compaction tuning, hybrid rings over ExpressRoute).
  - **Keyspaces** is serverless but restricts consistency levels and features. Test LWT, secondary indexes and TTL behaviour before migrating.
  - **MongoDB:** both clouds offer compatible engines (Amazon DocumentDB, Azure DocumentDB). For full MongoDB features (latest server, Atlas Search/Vector Search), use **MongoDB Atlas**, available on both via marketplace.
- **Search and logs (C6.41–C6.47):**
  - AWS has first-party OpenSearch.
  - Azure's first-party log store is **Log Analytics (KQL)**. Elasticsearch on Azure is Elastic's ISV service.
  - Pick **Azure AI Search** for application/RAG search, not for log pipelines.
  - Legacy Azure Log Analytics agent (MMA) is replaced by AMA + DCRs, and the HTTP Data Collector API by the Logs ingestion API.
- **Big data (C6.48–C6.51):**
  - S3 ↔ ADLS Gen2 replaces HDFS.
  - EMR ↔ Databricks/Fabric is the practical modern pair. Treat HDInsight as legacy (5.1 only, Spark 3.3).
  - Managed Flink ↔ Stream Analytics is the closest streaming pair, but ASA is not Flink. For portable Flink on Azure, use Confluent Cloud or Flink on AKS.

## Hands-on (optional)
```yaml
# docker compose: Nginx micro-cache in front of an app, plus Valkey and a single-node KRaft Kafka
services:
  nginx:
    image: nginx:1.29
    ports: ["8080:80"]
    volumes: ["./nginx.conf:/etc/nginx/conf.d/default.conf:ro"]
  app:
    image: traefik/whoami
  valkey:
    image: valkey/valkey:9
    command: ["valkey-server", "--appendonly", "yes", "--appendfsync", "everysec", "--maxmemory", "256mb", "--maxmemory-policy", "allkeys-lfu"]
  kafka:
    image: apache/kafka:4.1.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

```bash
# nginx.conf micro-cache (write before `docker compose up`)
cat > nginx.conf <<'EOF'
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=micro:10m max_size=1g inactive=10m;
upstream app { server app:80; keepalive 32; }
server {
  listen 80;
  location / {
    proxy_pass http://app;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_cache micro;
    proxy_cache_valid 200 5s;
    proxy_cache_lock on;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503;
    proxy_cache_background_update on;
    add_header X-Cache $upstream_cache_status;
  }
}
EOF
docker compose up -d
curl -sI localhost:8080 | grep X-Cache   # MISS then HIT

# Apache: which MPM is active?
apachectl -V | grep -i mpm

# Redis/Valkey: persistence + slot for a key
docker compose exec valkey valkey-cli INFO persistence | grep -E 'aof_enabled|rdb_last_bgsave_status'
docker compose exec valkey valkey-cli CLUSTER KEYSLOT '{user:1}:cart'   # works without cluster mode

# Kafka: create a topic, inspect ISR
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server kafka:9092 \
  --create --topic orders --partitions 6 --replication-factor 1
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server kafka:9092 --describe --topic orders

# Node: increase libuv threadpool for fs/dns-heavy services
UV_THREADPOOL_SIZE=16 node server.js
```

```hcl
# Terraform: SQS with DLQ (at-least-once + poison handling) and an ElastiCache Valkey serverless cache
resource "aws_sqs_queue" "dlq" {
  name                      = "orders-dlq"
  message_retention_seconds = 1209600 # 14 days
}

resource "aws_sqs_queue" "orders" {
  name                       = "orders"
  visibility_timeout_seconds = 60
  receive_wait_time_seconds  = 20 # long polling
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 5
  })
}

resource "aws_elasticache_serverless_cache" "cache" {
  engine = "valkey"
  name   = "app-cache"
  cache_usage_limits {
    data_storage {
      maximum = 10
      unit    = "GB"
    }
  }
}
```

## Cross-links
- [C1 Performance: caching (C1.26–C1.30)](../C-large-scale-architecture/C1-performance.md#c127-caching-for-performance)
- [C2 Scalability: stateless tiers, partitioning, L4/L7 LB](../C-large-scale-architecture/C2-scalability.md)
- [C3 Reliability: DR, standby](../C-large-scale-architecture/C3-reliability.md)
- [C5 Deployment: blue/green, slots](../C-large-scale-architecture/C5-deployment.md)
- [D1 System design basics: CDN (D1.14), caching (D1.15), LB (D1.3–D1.5), consistent hashing (D1.21)](../D-system-design/D1-system-design-basics.md)
- [D2 Reusable parts of system design](../D-system-design/D2-reusable-parts-of-system-design.md)
- [B6 Database sharding: consistent hashing (B6.2)](../B-database-engineering/B6-database-sharding.md)
- [A7 Socket management: epoll, accept queues](../A-operating-systems/A7-socket-management.md)
- [H6 Web application architecture: proxies, CDN (H6.4–H6.12)](../H-full-stack-troubleshooting/H6-web-application-architecture.md)
- [I3 Acceleration (CDN, edge)](../I-dns-tls-acceleration-gaps/I3-acceleration.md)
- [M4 Kafka at scale](../M-data-platforms/M4-kafka-at-scale.md) · [M5 Stream processing](../M-data-platforms/M5-stream-processing.md)
- [K7 AI gateways, caching, cost](../K-ai-infra-llm/K7-ai-gateways-caching-cost.md)
- [J5 Capacity planning and load testing](../J-sre/J5-capacity-planning-load-testing.md)
- [B1 ACID](../B-database-engineering/B1-acid.md) · [B2 Database internals](../B-database-engineering/B2-database-internals.md) · [B4 B-tree vs B+tree](../B-database-engineering/B4-btree-vs-bplustree.md)
- [B5 Partitioning](../B-database-engineering/B5-database-partitioning.md) · [B6 Sharding](../B-database-engineering/B6-database-sharding.md) · [B8 Replication](../B-database-engineering/B8-database-replication.md) · [B9 Database system design](../B-database-engineering/B9-database-system-design.md) · [B10 Database engines](../B-database-engineering/B10-database-engines.md)
- [C2 Scalability: partitioning/sharding (C2.18–C2.20), replication (C2.10–C2.11)](../C-large-scale-architecture/C2-scalability.md)
- [D1 System design basics: partitioning (D1.22)](../D-system-design/D1-system-design-basics.md)
- [J3 Observability (log pipelines)](../J-sre/J3-observability.md)
- [K2 Embeddings and vector databases](../K-ai-infra-llm/K2-embeddings-vector-databases.md)
- [M1 Lakehouse table formats](../M-data-platforms/M1-lakehouse-table-formats.md) · [M2 Spark at scale](../M-data-platforms/M2-spark-at-scale.md) · [M3 Databricks](../M-data-platforms/M3-databricks-platform.md) · [M6 Orchestration/ETL](../M-data-platforms/M6-orchestration-etl.md) · [M7 Data warehouses](../M-data-platforms/M7-data-warehouses.md)

## Sources
- https://httpd.apache.org/docs/2.4/mpm.html
- https://httpd.apache.org/docs/2.4/mod/event.html
- https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
- https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html
- https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/
- https://redis.io/blog/agplv3/
- https://valkey.io/blog/
- https://www.rabbitmq.com/docs/quorum-queues
- https://www.rabbitmq.com/docs/classic-queues
- https://kafka.apache.org/community/downloads/
- https://kafka.apache.org/blog/2025/09/04/apache-kafka-4.1.0-release-announcement/
- https://kafka.apache.org/blog/2026/02/17/apache-kafka-4.2.0-release-announcement/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html
- https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.html
- https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html
- https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/retirement-faq
- https://learn.microsoft.com/en-us/azure/frontdoor/front-door-cdn-comparison
- https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview
- https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas
- https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-quotas
- https://www.usenix.org/conference/atc22/presentation/elhemali (Amazon DynamoDB, USENIX ATC 2022: Multi-Paxos replication groups, 2/3 write quorum, log replicas)
- https://www.usenix.org/system/files/atc22-elhemali.pdf
- https://cassandra.apache.org/_/download.html
- https://cassandra.apache.org/doc/latest/cassandra/new/index.html
- https://docs.aws.amazon.com/keyspaces/latest/devguide/what-is-keyspaces.html
- https://docs.aws.amazon.com/keyspaces/latest/devguide/consistency.html
- https://learn.microsoft.com/en-us/azure/managed-instance-apache-cassandra/introduction
- https://learn.microsoft.com/en-us/azure/cosmos-db/cassandra/introduction
- https://learn.microsoft.com/en-us/azure/documentdb/overview
- https://www.mongodb.com/docs/manual/release-notes/
- https://docs.cloud.google.com/bigtable/docs/overview
- https://www.elastic.co/blog/elasticsearch-is-open-source-again
- https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards
- https://opensearch.org/releases/
- https://docs.fluentbit.io/manual/about/fluentd-and-fluent-bit
- https://spark.apache.org/news/index.html
- https://spark.apache.org/releases/spark-release-4-0-0.html
- https://docs.aws.amazon.com/managed-flink/latest/java/what-is.html
- https://docs.aws.amazon.com/kinesisanalytics/latest/dev/discontinuation.html
- https://learn.microsoft.com/en-us/azure/stream-analytics/stream-analytics-introduction
- https://learn.microsoft.com/en-us/azure/hdinsight/hdinsight-component-versioning
- https://learn.microsoft.com/en-us/azure/hdinsight/hdinsight-overview
- https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-rule-overview
