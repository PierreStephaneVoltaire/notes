# Infra, SRE & System Design Curriculum (Flashcard Source List)

Source list for generating one summary `.md` file per ID (bullet points + mermaid diagram).

## How this list works

- **One file per ID.** Every bolded ID (for example `C1.12`) is one flashcard topic. Nested bullets under an ID are that lecture's own sub points and belong inside that ID's file, not in separate files.
- **Cloud agnostic.** Vendor names were swapped for generic terms. Cloud provider context (AWS and Azure) gets added afterwards.
- **† marker:** the original lecture named a specific cloud service or covers cloud offerings. Add AWS and Azure context to these first. **All of section G is †.**
- **◆ marker:** not from a course page. These are gaps the courses don't cover.
- **Dropped from the raw course pages:** intros, summaries, quizzes, slides, exam essentials, environment setup and code walkthrough lectures, and non infra topics (Linux basics, email, SFTP, wireless).
- **Suggested study order:** A → F → H → B → G → C → D → E, with I alongside the related sections.

## Data caveats

- **B10 to B14** came from a third party listing because Udemy hides the last 8 sections of that course. The course now has 18 sections and was updated September 2026, so a section may be missing.
- **C6.32 and C6.33** teach the masterless, gossip based design from the original Dynamo paper. Today's managed DynamoDB works differently, so flag that in those files.
- **E** has 4 sections hidden on Udemy's page. One is likely the autonomous travel booking agent case study.
- **F** has 2 of 12 sections hidden on Udemy's page.
- **G15** lists sections named in the course description whose lectures could not be retrieved.

## Overlaps (cross link instead of repeating)

| Topic | IDs |
|---|---|
| Consistent hashing | B6.2, D1.21 |
| Partitioning and sharding | B5, B6, C2.18 to C2.20, D1.22 |
| Caching | C1.26 to C1.30, C2.16, C6.18 to C6.21, D1.14, D1.15, H6.4 |
| CDN | C6.15, D1.14, H6.12 |
| Replication | B8, C2.10, C2.11 |
| Locking and concurrency | B7, C1.20 to C1.24 |
| DR and standby | C3.24 to C3.26, D1.23, D1.24 |
| TLS and certificates | C4.5 to C4.11, D2.11, D2.12, F5.2, F5.3, H4 |
| Sockets and kernel queues | A7.1, F4.9, H6.7 |
| L4 vs L7 load balancing and proxies | C2.26, D1.3 to D1.5, F6.8, F6.9, H6.5, H6.6 |
| Firewalls and ACLs | C4.12, C4.13, D2.13, G1.7, G1.8, H2.7 |
| MTU | F6.1, G4.1, G12.27, H5.8 |
| DNS | F5.1, G3, H3 |
| Latency | C1.6 to C1.15, F6, H5 |
| Diagnostic tools | F2.3, F2.5, F8, H1, H2 |

## Courses

| Section | Course | Instructor |
|---|---|---|
| A | Fundamentals of Operating Systems | Hussein Nasser |
| B | Fundamentals of Database Engineering | Hussein Nasser |
| C | Software Architecture & Technology of Large-Scale Systems | Anurag Yadav |
| D | Rocking System Design | Rajdeep Saha |
| E | AI System Design for Engineers and Interviews (optional) | Aritra Basak |
| F | Fundamentals of Network Engineering | Hussein Nasser |
| G | AWS Certified Advanced Networking Specialty (neutralized) | Stephane Maarek, Chetan Agrawal |
| H | Full Stack Troubleshooting: Network, Linux & NetDevOps | Shaun Hummel |
| I | Gaps not covered by any course (◆) | n/a |

---

## A. Fundamentals of Operating Systems (Hussein Nasser)

### A1 Why an OS?
- **A1.1** Why do we need an Operating System?
- **A1.2** System Architecture Overview
  - CPU, memory, storage, network, file system, security
  - Program, process, kernel, userspace, kernel space (drivers), system calls

### A2 The Anatomy of a Process
- **A2.1** Program vs Process
- **A2.2** Simple Process Execution
- **A2.3** The Stack
- **A2.4** Process Execution with Stack
- **A2.5** Data section
- **A2.6** The Heap
- **A2.7** Process memory mappings (/proc/&lt;pid&gt;/maps)

### A3 Memory Management
- **A3.1** The Anatomy of Memory
- **A3.2** Reading and Writing from and to Memory
- **A3.3** Virtual Memory
  - Page tables and the cost of virtual memory
  - Internal vs external fragmentation
  - Shared memory, isolation, swap
- **A3.4** DMA (Direct Memory Access)
- **A3.5** Reading memory usage: virtual, resident, shared

### A4 Inside the CPU
- **A4.1** CPU Components and Architecture
  - ALU, control unit, registers and caches, clock speed, cores
  - Pipelining, parallelism, RISC vs CISC, 32 vs 64 bit
- **A4.2** Instruction Life Cycle (L1/L2/L3 lookups, 64 byte cache lines)
- **A4.3** Pipelining and Parallelism (hyper-threading / SMT)
- **A4.4** CPU wait times: IO-bound vs CPU-bound workloads

### A5 Process Management
- **A5.1** Process vs Thread
- **A5.2** Context Switching (PCB, page table swap, TLB flush)
- **A5.3** Concurrency (mutexes, semaphores)
- **A5.4** fork() and copy-on-write
- **A5.5** When do you use threads?

### A6 Storage Management
- **A6.1** Persistent Storage
  - Why persist?
  - HDD vs SSD
  - App → File System → Block Layer → NVMe Driver
- **A6.2** File Systems
  - FAT, ext4
  - Page cache
  - Read vs write path
  - File modes: O_SYNC, O_DIRECT
- **A6.3** What really happens in a file IO? (page cache, LBA vs PBA, block mapping)
- **A6.4** Partitioning, formatting and mounting file systems (ext4, ZFS)

### A7 Socket Management
- **A7.1** Sockets, Connections and Kernel Queues
  - SYN queue vs accept queue
  - Receive vs send queue
  - Kernel memory for connection lookup
- **A7.2** Sending and Receiving Data
- **A7.3** Socket Programming Patterns (listener, acceptor, reader, parser, decoder)
- **A7.4** Asynchronous IO: select, epoll, IOCP, kqueue, io_uring

### A8 More OS Concepts
- **A8.1** Compilers and Linkers
- **A8.2** Kernel vs User Mode switching
- **A8.3** Virtualization and Containerization (cgroups vs namespaces)

### A9 Bonus
- **A9.1** How Google Improved Linux TCP/IP Stack by 40%
- **A9.2** How TikTok's Bytedance improved Linux reboot
- **A9.3** On Timeouts
- **A9.4** Postgres failure caused by a Cisco router (TCP issue)
- **A9.5** Page Tables

---

## B. Fundamentals of Database Engineering (Hussein Nasser)

### B1 ACID
- **B1.1** What is a Transaction?
- **B1.2** Atomicity
- **B1.3** Isolation (dirty, non-repeatable and phantom reads, lost updates, isolation levels)
- **B1.4** Consistency
- **B1.5** Durability (WAL, fsync)
- **B1.6** Phantom Reads
- **B1.7** Serializable vs Repeatable Read
- **B1.8** Eventual Consistency

### B2 Understanding Database Internals
- **B2.1** How tables and indexes are stored on disk
- **B2.2** Row-Based vs Column-Based Databases
- **B2.3** Primary Key vs Secondary Key
- **B2.4** Database Pages

### B3 Database Indexing
- **B3.1** Getting Started with Indexing
- **B3.2** Understanding the SQL Query Planner and Optimizer with Explain
- **B3.3** Bitmap Index Scan vs Index Scan vs Table Scan
- **B3.4** Key vs Non-Key Column Database Indexing
- **B3.5** Index Scan vs Index Only Scan
- **B3.6** Combining Database Indexes for Better Performance
- **B3.7** How Database Optimizers Decide to Use Indexes
- **B3.8** Create Index Concurrently: Avoid Blocking Production Database Writes
- **B3.9** Bloom Filters
- **B3.10** Working with Billion-Row Table
- **B3.11** How UUIDs in B+Tree Indexes affect performance
- **B3.12** The Cost of Long running Transactions
- **B3.13** Clustered Index Design

### B4 B-Tree vs B+Tree in Production Database Systems
- **B4.1** Full Table Scans
- **B4.2** Original B-Tree
- **B4.3** How the Original B-Tree Helps Performance
- **B4.4** Original B-Tree Limitations
- **B4.5** B+Tree
- **B4.6** B+Tree DBMS Considerations
- **B4.7** B+Tree Storage Cost in MySQL vs Postgres

### B5 Database Partitioning
- **B5.1** What is Partitioning?
- **B5.2** Vertical vs Horizontal Partitioning
- **B5.3** Partitioning Types
- **B5.4** The Difference Between Partitioning and Sharding
- **B5.5** Partition pruning
- **B5.6** The Advantages of Partitioning
- **B5.7** The Disadvantages of Partitioning
- **B5.8** How to Automate Partitioning

### B6 Database Sharding
- **B6.1** What is Database Sharding?
- **B6.2** Consistent Hashing
- **B6.3** Horizontal partitioning vs Sharding
- **B6.4** Writing to a Shard
- **B6.5** Reading from a Shard
- **B6.6** Advantages of Database Sharding
- **B6.7** Disadvantages of Database Sharding
- **B6.8** When Should you consider Sharding your Database?

### B7 Concurrency Control
- **B7.1** Shared vs Exclusive Locks
- **B7.2** Dead Locks
- **B7.3** Two-phase Locking
- **B7.4** Solving the Double Booking Problem
- **B7.5** Double Booking Problem Part 2 (alternative solution)
- **B7.6** SQL Pagination With Offset is Very Slow
- **B7.7** Database Connection Pooling

### B8 Database Replication
- **B8.1** Master/Standby Replication
- **B8.2** Multi-master Replication
- **B8.3** Synchronous vs Asynchronous Replication
- **B8.4** Pros and Cons of Replication

### B9 Database System Design
- **B9.1** Twitter System Design Database Design
- **B9.2** Building a Short URL System Database Backend

### B10 Database Engines
- **B10.1** What is a Database Engine?
- **B10.2** MyISAM
- **B10.3** InnoDB
- **B10.4** XtraDB
- **B10.5** SQLite
- **B10.6** Aria
- **B10.7** BerkeleyDB
- **B10.8** LevelDB
- **B10.9** RocksDB
- **B10.10** Popular Database Engines
- **B10.11** Switching Database Engines with MySQL

### B11 Database Cursors
- **B11.1** What are Database Cursors?
- **B11.2** Server Side vs Client Side Database Cursors
- **B11.3** Querying with Client Side Cursor
- **B11.4** Querying with Server Side Cursor
- **B11.5** Pros and Cons of Server vs Client Side Cursors

### B12 Database Security
- **B12.1** How to Secure Your Postgres Database by Enabling TLS/SSL
- **B12.2** What is the Largest SQL Statement that You can Send to Your Database
- **B12.3** Deep Look into Postgres Wire Protocol with Wireshark
- **B12.4** Deep Look Into MongoDB Wire Protocol with Wireshark
- **B12.5** Best Practices Working with REST & Databases
- **B12.6** Database Permissions and Best Practices for Building REST API

### B13 Homomorphic Encryption: Performing Database Queries on Encrypted Data
- **B13.1** What is Encryption?
- **B13.2** Why Can't we always Encrypt?
- **B13.3** What is Homomorphic Encryption
- **B13.4** Searching The Encrypted Database
- **B13.5** Is Homomorphic Encryption Ready?

### B14 Database Discussions
- **B14.1** SELECT COUNT(*) can impact your Backend Application performance
- **B14.2** How does the Database Store Data On Disk?
- **B14.3** Is QUIC a Good Protocol for Databases?
- **B14.4** What is a Distributed Transaction?
- **B14.5** Why Uber Moved from Postgres to MySQL
- **B14.6** Can NULLs Improve your Database Queries Performance?
- **B14.7** Write Amplification Explained in Backend Apps, Database Systems and SSDs
- **B14.8** Optimistic vs Pessimistic Concurrency Control

---

## C. Software Architecture & Technology of Large-Scale Systems (Anurag Yadav)

### C1 Performance
- **C1.1** What is performance
- **C1.2** How do performance problems look like
- **C1.3** Performance principles
- **C1.4** System performance objectives
- **C1.5** Performance measurement metrics
- **C1.6** Serial request latency
- **C1.7** Network transfer latency
- **C1.8** Minimizing network transfer latency
- **C1.9** Memory access latency
- **C1.10** Minimizing memory access latency
- **C1.11** Disk access latency
- **C1.12** Minimizing disk access latency
- **C1.13** CPU processing latency
- **C1.14** Minimizing CPU processing latency
- **C1.15** Some common latency costs
- **C1.16** Amdahl's law for concurrent tasks
- **C1.17** Gunther's universal scalability law
- **C1.18** Shared resource contention
- **C1.19** Minimizing shared resource contention
- **C1.20** Minimizing locking related contention
- **C1.21** Pessimistic Locking
- **C1.22** Optimistic Locking
- **C1.23** Compare and swap mechanism
- **C1.24** Deadlocks
- **C1.25** Coherence related delays
- **C1.26** System architecture for performance
- **C1.27** Caching for performance
- **C1.28** HTTP Caching of static data
- **C1.29** Caching of dynamic data
- **C1.30** Caching related challenges

### C2 Scalability
- **C2.1** Performance vs Scalability
- **C2.2** Vertical & Horizontal scalability
- **C2.3** Reverse proxy
- **C2.4** Scalability principles
- **C2.5** Modularity for scalability
- **C2.6** Replication
- **C2.7** Stateful replication in web applications
- **C2.8** Stateless replication in web applications
- **C2.9** Stateless replication of services
- **C2.10** Database replication
- **C2.11** Database replication types
- **C2.12** Need for specialized services
- **C2.13** Specialized services: SOAP/REST
- **C2.14** Asynchronous services
- **C2.15** Asynchronous processing & scalability
- **C2.16** Caching for scalability
- **C2.17** Vertical partitioning with micro-services
- **C2.18** Database partitioning
- **C2.19** Database partitioning selection
- **C2.20** Routing with database partitioning
- **C2.21** Methods for horizontal scalability
- **C2.22** Load balancing multiple instances
- **C2.23** Discovery service and load balancing
- **C2.24** Load balancer discovery
- **C2.25** HLB vs SLB (hardware vs software load balancers)
- **C2.26** Layer-7 load balancers
- **C2.27** DNS as load balancer
- **C2.28** Global server load balancing
- **C2.29** Global data replication
- **C2.30** Auto scaling instances
- **C2.31** Micro-Services Motivation
- **C2.32** Service Oriented Architecture
- **C2.33** Micro-Services Architecture Style
- **C2.34** Transactions in Micro-Services
- **C2.35** Compensating Transactions: SAGA Pattern
- **C2.36** Micro-services communication model
- **C2.37** Event driven transactions
- **C2.38** Extreme scalability with NoSQL and Kafka

### C3 Reliability
- **C3.1** Failures in large scale distributed systems
- **C3.2** Partial system failures
- **C3.3** Reliability
- **C3.4** Availability
- **C3.5** High Availability
- **C3.6** Fault Tolerance
- **C3.7** Fault tolerant design
- **C3.8** Redundancy
- **C3.9** Types of redundancy
- **C3.10** Single point of failures
- **C3.11** Stateless component redundancy
- **C3.12** Stateful component redundancy
- **C3.13** Load balancer redundancy
- **C3.14** Datacentre infrastructure as SPOF
- **C3.15** Creating datacenter redundancy
- **C3.16** Fault models
- **C3.17** Health checks
- **C3.18** External monitoring service
- **C3.19** Internal cluster monitoring
- **C3.20** Fault detection in a system
- **C3.21** Stateless component recovery
- **C3.22** Stateful Failovers
- **C3.23** Load Balancer high availability
- **C3.24** Database recovery with hot standby
- **C3.25** Database recovery with warm standby
- **C3.26** Database recovery with cold backups
- **C3.27** High Availability in large scale systems
- **C3.28** Failover best practices
- **C3.29** Timeouts
- **C3.30** Retries
- **C3.31** Circuit Breaker
- **C3.32** Fail Fast and Shed Load

### C4 Security
- **C4.1** Security objectives
- **C4.2** Symmetric key encryption
- **C4.3** Public key encryption
- **C4.4** Secure network protocol
- **C4.5** SSL and TLS
- **C4.6** Hashing
- **C4.7** Digital signatures
- **C4.8** Digital certificates
- **C4.9** Chain of trust
- **C4.10** TLS/SSL handshake
- **C4.11** Secure network channel
- **C4.12** Firewalls
- **C4.13** Network security (subnets, DMZ, port rules)
- **C4.14** Authentication and authorization
- **C4.15** Credentials transfer
- **C4.16** Credentials verification
- **C4.17** Stateful authentication
- **C4.18** Stateless authentication
- **C4.19** Single Sign-On
- **C4.20** Role based access control model
- **C4.21** Role based access example
- **C4.22** Authorization
- **C4.23** OAuth2 token grant
- **C4.24** OAuth2 token grant: Code Flow
- **C4.25** OAuth2 token grant: Password Flow
- **C4.26** OAuth2 in a system
- **C4.27** OAuth2 token types
- **C4.28** JSON Web Tokens
- **C4.29** Token storage
- **C4.30** Securing data at rest
- **C4.31** Securing a Software System
- **C4.32** SQL Injection
- **C4.33** Cross Site Scripting
- **C4.34** Cross Site Request Forgery

### C5 Deployment
- **C5.1** Large scale deployment challenges
- **C5.2** Application deployment
- **C5.3** Infrastructure deployment
- **C5.4** System operations
- **C5.5** Modern deployment solutions
- **C5.6** Component deployment
- **C5.7** Component deployment automation
- **C5.8** Deployment with Virtual Machines
- **C5.9** Isolation through virtual machines
- **C5.10** Deployment with Containers
- **C5.11** Docker containers
- **C5.12** Infrastructure requirements
- **C5.13** Provisioning and configuration
- **C5.14** Deployment with containers on Cloud †
- **C5.15** Deployment with a cloud provider stack †
- **C5.16** Kubernetes lifecycle management
- **C5.17** Kubernetes naming and addressing
- **C5.18** Kubernetes scaling with multiple instances
- **C5.19** Kubernetes load balancing
- **C5.20** Kubernetes high availability
- **C5.21** Kubernetes rolling upgrades
- **C5.22** Kubernetes capabilities
- **C5.23** Kubernetes deployment
- **C5.24** Kubernetes services and workloads
- **C5.25** Kubernetes architecture
- **C5.26** Rolling updates
- **C5.27** Canary deployment
- **C5.28** Recreate deployment
- **C5.29** Blue Green deployment
- **C5.30** A/B testing

### C6 Technology Stack
- **C6.1** Web applications
- **C6.2** Solutions for web applications
- **C6.3** Apache web server
- **C6.4** Apache webserver architecture
- **C6.5** Apache webserver scalability
- **C6.6** Nginx webserver
- **C6.7** Nginx architecture
- **C6.8** Nginx as reverse proxy and cache
- **C6.9** Web containers & Spring framework
- **C6.10** Jetty & Spring
- **C6.11** Node.js
- **C6.12** Node.js event loop
- **C6.13** Cloud solutions for web †
- **C6.14** Cloud storage †
- **C6.15** Cloud CDN †
- **C6.16** Services
- **C6.17** Services solutions
- **C6.18** Memcached
- **C6.19** Memcached Architecture
- **C6.20** Redis Cache & its architecture
- **C6.21** Cloud caching solutions †
- **C6.22** RabbitMQ
- **C6.23** RabbitMQ architecture
- **C6.24** Kafka architecture
- **C6.25** Redis Pub/Sub
- **C6.26** Cloud MQ solutions †
- **C6.27** Datastores
- **C6.28** Datastore solutions
- **C6.29** RDBMS
- **C6.30** RDBMS scalability architecture
- **C6.31** NoSQL objectives & trade-offs
- **C6.32** Leaderless key-value store (Dynamo model) †
- **C6.33** Dynamo architecture: consistent hashing and gossip †
- **C6.34** BigTable wide-column model †
- **C6.35** BigTable architecture
- **C6.36** HBase
- **C6.37** Cassandra
- **C6.38** Cassandra features
- **C6.39** MongoDB
- **C6.40** MongoDB architecture
- **C6.41** Analytics
- **C6.42** Analytics solutions
- **C6.43** Logstash architecture
- **C6.44** Logstash data streaming architecture
- **C6.45** Fluentd
- **C6.46** Elasticsearch
- **C6.47** Elasticsearch architecture
- **C6.48** Hadoop HDFS
- **C6.49** Map-Reduce
- **C6.50** Apache Spark
- **C6.51** Stream processing

---

## D. Rocking System Design (Rajdeep Saha)

### D1 System Design Basics
- **D1.1** Monolith vs Microservices: What and Why
- **D1.2** Microservices on a cloud platform †
- **D1.3** Load Balancing: L7 vs L4 load balancers †
- **D1.4** API and API Gateway
- **D1.5** Load Balancer vs API Gateway †
- **D1.6** Scaling: Vertical vs Horizontal
- **D1.7** VM, Serverless, Container Scaling
- **D1.8** Real World Scaling Interview Tips
- **D1.9** Synchronous vs Event Driven Architectures
- **D1.10** Queues vs PubSub
- **D1.11** Streaming vs Messaging
- **D1.12** SQL vs NoSQL, and managed distributed relational vs managed key-value NoSQL †
- **D1.13** WebSockets for Server to Client Communication
- **D1.14** Caching (where to cache: gateway, CDN, cache cluster; TTL)
- **D1.15** Redis and Memcached Caching Strategies
- **D1.16** High Availability
- **D1.17** High Availability vs Fault Tolerance
- **D1.18** Distributed Computing
- **D1.19** Hashing
- **D1.20** Challenges of Hashing
- **D1.21** Consistent Hashing
- **D1.22** Database Sharding
- **D1.23** Disaster Recovery: RPO vs RTO
- **D1.24** Different Disaster Recovery Options
  - Backup and restore, pilot light, warm standby, multi-site active-active
- **D1.25** CAP Theorem

### D2 Reusable Parts of System Design
- **D2.1** Well-Architected Framework †
- **D2.2** Three-Tier Architecture
- **D2.3** Three-Tier Architecture on Serverless and Kubernetes
- **D2.4** Content Based Messaging System
- **D2.5** Store and Retrieve Images
- **D2.6** High Priority Queuing/Messaging System
- **D2.7** Data Analytics & Big Data Design Patterns
- **D2.8** Performance and Cost Optimization
- **D2.9** Security: Authentication (Log In) & Authorization
- **D2.10** Security: Encryption at Rest & Client/Server Side Encryption (envelope encryption, data keys vs master keys)
- **D2.11** Security: Encryption In Transit with SSL/TLS/mTLS
- **D2.12** TLS vs mTLS
- **D2.13** IDS vs IPS vs instance firewalls and network ACLs †
- **D2.14** Identity and access management: users, roles, groups †
- **D2.15** Twelve Factor App
- **D2.16** Cell Based Architecture

### D3 System Design of Modern Applications
- **D3.1** Must-knows for system design
  - Microservices behind a load balancer or API gateway
  - Sync vs async patterns
  - Database selection and optimization
  - Caching
  - Security
- **D3.2** Design YouTube/Netflix/Prime Video
  - Requirements and features
  - Video ingestion
  - Database
  - Video encoding
  - Adult content detection
  - Parallel processing
  - Content Delivery Network (CDN)
  - Searching and viewing video
  - Object storage cost savings techniques †
  - Security
- **D3.3** Design Twitter
  - Features and design requirements
  - Table design
  - Timeline design
  - Architecture on a cloud platform †
  - Database discussion
  - Edge case
  - Trending feature
  - Security
- **D3.4** Design WhatsApp/Telegram/Snapchat
  - Message queue, WebSocket API, online status checks
- **D3.5** Design Tinder
  - Requirements and design specs
  - Image store/retrieve design
  - Match search and database design
  - Recommendation engine
  - Precomputing matches
  - Chat feature
- **D3.6** Design Uber
  - Schemaless (Uber's MySQL-based datastore)
  - Ringpop (hash ring and gossip membership)
- **D3.7** Design Fandango/Ticketmaster/Livenation
  - Separating ticket transactions from catalog data
  - Seat locking with timed holds
- **D3.8** IoT System Design
  - MQTT broker and topics, edge computing, IoT analytics
- **D3.9** Design Shopify
  - Requirements and design spec
  - Online store design
  - Cost effectiveness and scaling
  - Database design
  - Security
  - Analytics and high availability
- **D3.10** Design URL Shortener/TinyURL
- **D3.11** Design Parking Garage
- **D3.12** Design Amazon.com/Flipkart
- **D3.13** Design Gen AI Systems

---

## E. AI System Design for Engineers and Interviews (Aritra Basak), optional

### E1 System Design Fundamentals
- **E1.1** Introduction to System Design (traditional vs AI system design)
- **E1.2** Core Principles of System Design (scalability, availability, reliability, performance)
- **E1.3** Introduction to AI System Design
- **E1.4** Databases in System Design (cache miss, cache breakdown, sharding)
- **E1.5** Monolithic Architecture Fundamentals
- **E1.6** Microservices Architecture Fundamentals
- **E1.7** Containers and Modern System Design (images, registries, volumes, networks, GitOps)
- **E1.8** Core Components of Modern System Design
- **E1.9** RAG Chunking and Retrieval Strategies
- **E1.10** GraphRAG: Knowledge Graphs for Advanced Retrieval

### E2 Google CTR Prediction System Case Study
- **E2.1** Problem Overview (data drift, bias, model drift)
- **E2.2** Gathering Requirements
- **E2.3** Data Pipeline & Model Training
- **E2.4** Inference Architecture

### E3 HubSpot User Clustering Case Study
- **E3.1** Requirements & Design
- **E3.2** System Workflow (batch vs online learning, ETL, silver and gold layers, nightly retraining)
- **E3.3** Training & Inference Architecture

### E4 Facebook Content Moderation Case Study
- **E4.1** System Requirements
- **E4.2** Architecture & Model Selection
- **E4.3** Training & Inference Pipeline

### E5 Smart Car Parking SaaS Case Study (computer vision)
- **E5.1** Requirements Analysis
- **E5.2** Architecture Design

### E6 AI Grammar Checker SaaS Case Study
- **E6.1** System Planning & Requirements (100k concurrent users, sub-200 ms P99)
- **E6.2** Model Training Architecture
- **E6.3** Inference Architecture

### E7 AI Interview Chatbot Case Study
- **E7.1** System Requirements
- **E7.2** Training & Inference Pipeline (fine-tuning, reward model, RLHF)
- **E7.3** System Architecture

### E8 Deep Research Agent Case Study
- **E8.1** System Planning
- **E8.2** Multi-Agent Architecture

---

## F. Fundamentals of Network Engineering (Hussein Nasser)

### F1 Fundamentals of Networking
- **F1.1** Client-Server Architecture
- **F1.2** OSI Model (where applications, proxies and network devices sit in each layer)
- **F1.3** Host to Host communication (MAC, IP, subnet masks, routers, gateways, ports)

### F2 Internet Protocol (IP)
- **F2.1** The IP Building Blocks
- **F2.2** IP Packet (header, MTU, fragmentation, TTL)
- **F2.3** ICMP, PING, TraceRoute
- **F2.4** ARP
- **F2.5** Capturing IP, ARP and ICMP Packets with TCPDUMP
- **F2.6** Routing Example
- **F2.7** Private IP addresses (RFC 1918)

### F3 User Datagram Protocol (UDP)
- **F3.1** What Is UDP?
- **F3.2** User Datagram Structure
- **F3.3** UDP Pros & Cons
- **F3.4** Capturing UDP traffic with TCPDUMP

### F4 Transmission Control Protocol (TCP)
- **F4.1** What is TCP?
- **F4.2** TCP Segment
- **F4.3** Flow Control
- **F4.4** Congestion Control
- **F4.5** Slow Start vs Congestion Avoidance
- **F4.6** NAT
- **F4.7** TCP Connection States
- **F4.8** TCP Pros and Cons
- **F4.9** Sockets, Connections and Kernel Queues
- **F4.10** Capturing TCP Segments with TCPDUMP

### F5 Overview of Popular Networking Protocols
- **F5.1** DNS
- **F5.2** TLS
- **F5.3** HTTPS, TLS, Keys and Certificates

### F6 Network Performance
- **F6.1** MSS vs MTU vs PMTUD
- **F6.2** Nagle's Algorithm's Effect on Performance
- **F6.3** Delayed Acknowledgment Effect on Performance
- **F6.4** Cost of Connection Establishment
- **F6.5** TCP Fast Open
- **F6.6** Listening Server
- **F6.7** TCP Head of line blocking
- **F6.8** The importance of Proxy and Reverse Proxies
- **F6.9** Load Balancing at Layer 4 vs Layer 7
- **F6.10** Network Access Control to Database Servers

### F7 Network Routing
- **F7.1** Fundamentals of Network Routing
- **F7.2** Networking with Docker (bridge networks, container DNS, gateways)

### F8 Analyzing Protocols with Wireshark
- **F8.1** Wiresharking UDP
- **F8.2** Wiresharking TCP/HTTP
- **F8.3** Wiresharking HTTP/2 (Decrypting TLS)
- **F8.4** Wiresharking MongoDB
- **F8.5** Wiresharking Server Sent Events

### F9 Answering your Questions
- **F9.1** Should Layer 4 Proxies buffer segments?
- **F9.2** How does the Kernel manage TCP connections?

---

## G. Cloud Network Architecture (from Maarek & Agrawal, all †)

### G1 Virtual Network Fundamentals
- **G1.1** What is a virtual network?
- **G1.2** Scope of a virtual network: account, region, availability zone
- **G1.3** Building blocks: CIDR, subnets, route tables, internet gateway, firewalls, DNS
- **G1.4** Addressing (CIDR)
- **G1.5** Route tables
- **G1.6** IP addresses: IPv4 vs IPv6, private vs public vs static public IP
- **G1.7** Instance-level stateful firewall rules
- **G1.8** Subnet-level stateless network ACLs
- **G1.9** Default virtual network
- **G1.10** Public vs private subnets
- **G1.11** Managed NAT gateway
- **G1.12** NAT gateway high availability
- **G1.13** Self-managed NAT instance
- **G1.14** Regional (multi-zone) NAT gateway

### G2 Additional Virtual Network Features
- **G2.1** IPv6 egress-only internet gateway
- **G2.2** Extending the address space (secondary CIDRs)
- **G2.3** Virtual network interfaces deep dive
- **G2.4** Bring your own IP

### G3 Network DNS and DHCP
- **G3.1** How DNS works
- **G3.2** Provider-managed DNS resolver inside the network
- **G3.3** DHCP option sets
- **G3.4** Private DNS zones for internal names
- **G3.5** Running a custom DNS server inside the network
- **G3.6** Hybrid DNS: inbound and outbound resolver endpoints

### G4 Network Performance and Optimization
- **G4.1** Basics of network performance: bandwidth, latency, jitter, throughput, PPS, MTU
- **G4.2** Placement groups and dedicated block-storage bandwidth
- **G4.3** Enhanced (SR-IOV) networking
- **G4.4** DPDK and kernel-bypass network adapters
- **G4.5** Bandwidth limits inside and outside the virtual network
- **G4.6** Burstable network bandwidth credits

### G5 Traffic Monitoring, Troubleshooting & Analysis
- **G5.1** Network flow logs
- **G5.2** Traffic (packet) mirroring
- **G5.3** Reachability analysis (hop-by-hop path)
- **G5.4** Network access compliance analysis

### G6 Private Connectivity: Peering
- **G6.1** Private connectivity options
- **G6.2** Virtual network peering
- **G6.3** Peering across regions
- **G6.4** Peering invalid scenarios (no transitive routing)

### G7 Private Connectivity: Service Endpoints & Private Link
- **G7.1** Introduction to service endpoints and private link
- **G7.2** Gateway (route-table based) endpoints
- **G7.3** Private-link interface endpoints
- **G7.4** Private link: important to know features
- **G7.5** Endpoint DNS
- **G7.6** Exposing your own service through private link
- **G7.7** Endpoint service with a custom domain name
- **G7.8** Endpoint security
- **G7.9** Other endpoint types: gateway load balancer, resource, service network
- **G7.10** Endpoint architectures: accessing on-premises services
- **G7.11** Endpoint architectures: accessing from peered or connected networks
- **G7.12** Endpoint architectures: accessing from on-premises
- **G7.13** Endpoint architectures: centralized endpoints
- **G7.14** Peering vs endpoints

### G8 Transit Hub
- **G8.1** Introduction to the transit hub router
- **G8.2** Network attachments and routing
- **G8.3** Full routing vs restricted (VRF-style) route tables
- **G8.4** Network patterns: flat vs segmented
- **G8.5** Availability zone considerations
- **G8.6** AZ affinity and appliance mode (symmetric routing)
- **G8.7** Hub peering across regions
- **G8.8** Connect attachment for SD-WAN appliances
- **G8.9** VPN attachment (ECMP, accelerated VPN)
- **G8.10** Hub and dedicated interconnect
- **G8.11** Multicast
- **G8.12** Architecture: centralized internet egress
- **G8.13** Architecture: centralized inspection with gateway load balancer
- **G8.14** Architecture: centralized inspection with managed network firewall
- **G8.15** Architecture: centralized inspection with network function attachment
- **G8.16** Architecture: centralized interface endpoints
- **G8.17** Transit hub vs peering
- **G8.18** Sharing the hub across accounts

### G9 Hybrid Network Basics
- **G9.1** Introduction to hybrid networking (IPsec VPN vs dedicated interconnect)
- **G9.2** Static vs dynamic routing
- **G9.3** How BGP works
- **G9.4** BGP route selection: AS_PATH, LOCAL_PREF, MED

### G10 Site-to-Site VPN
- **G10.1** Managed site-to-site VPN components
- **G10.2** IPv4 and IPv6 traffic over VPN
- **G10.3** Accelerated site-to-site VPN (anycast edge)
- **G10.4** VPN NAT traversal (NAT-T)
- **G10.5** VPN route propagation (static vs dynamic)
- **G10.6** VPN transitive routing scenarios
- **G10.7** VPN tunnels: active/active vs active/passive
- **G10.8** Dead Peer Detection (DPD)
- **G10.9** VPN monitoring
- **G10.10** Site-to-site VPN architectures
- **G10.11** VPN hub for branch offices
- **G10.12** Self-managed VPN appliances
- **G10.13** Transit network built from VPN appliances

### G11 Client VPN
- **G11.1** Managed remote-access VPN
- **G11.2** Split tunnel and remote network access

### G12 Dedicated Interconnect
- **G12.1** Introduction to dedicated interconnect
- **G12.2** Network requirements (physical layer up to BGP)
- **G12.3** BGP Autonomous System and ASN
- **G12.4** Connection types: dedicated vs hosted
- **G12.5** Steps to provision a connection
- **G12.6** Virtual interfaces: public, private, transit
- **G12.7** Virtual interface creation parameters
- **G12.8** Provider-published IP ranges
- **G12.9** Public virtual interface
- **G12.10** Private virtual interface
- **G12.11** Transit virtual interface
- **G12.12** Interconnect gateway with private interfaces
- **G12.13** Interconnect with transit hub
- **G12.14** Site-to-site routing over the provider backbone
- **G12.15** Routing policies and BGP communities
- **G12.16** Public interface routing policies
- **G12.17** Public interface routing scenarios (active/active, active/passive, ECMP)
- **G12.18** Public interface BGP communities
- **G12.19** Private interface routing policies and BGP communities
- **G12.20** Link aggregation groups (LAGs)
- **G12.21** Connection resiliency
- **G12.22** Failure detection with BFD
- **G12.23** Encrypting interconnect traffic
- **G12.24** Public IP VPN over interconnect (Layer 3)
- **G12.25** Private IP VPN over interconnect (Layer 3)
- **G12.26** MACsec encryption (Layer 2)
- **G12.27** MTU and jumbo frames
- **G12.28** Pricing
- **G12.29** Monitoring
- **G12.30** Troubleshooting Layer 1 to 4
- **G12.31** Architecture: putting it together

### G13 Managed Global WAN
- **G13.1** What is a managed global WAN?
- **G13.2** Core network policy
- **G13.3** Connecting transit hub and interconnect
- **G13.4** Traffic inspection: inspection network attachment
- **G13.5** Traffic inspection: network function groups (service insertion)

### G14 Service-to-Service Application Networking
- **G14.1** Introduction
- **G14.2** Components: service network, service, resource
- **G14.3** Network associations
- **G14.4** Traffic flow
- **G14.5** Service access with custom domain name
- **G14.6** Features: good to know

### G15 Sections listed in the course description (lectures not retrieved)
- **G15.1** Networking aspects of load balancers
- **G15.2** Networking aspects of the CDN
- **G15.3** Advanced DNS configurations
- **G15.4** Kubernetes networking
- **G15.5** Advanced network architectures
- **G15.6** Security services: DDoS protection, certificate management, firewall manager
- **G15.7** IP address management (IPAM)
- **G15.8** Shared virtual networks

---

## H. Full Stack Troubleshooting: Network, Linux & NetDevOps (Shaun Hummel)

### H1 Linux Network Diagnostics
- **H1.1** Interface Status
- **H1.2** IP Neighbor
- **H1.3** IP Addressing
- **H1.4** TCP Port Testing
- **H1.5** cURL Command

### H2 Troubleshooting Your Network
- **H2.1** Network Connection Basics (five-layer TCP/IP model, ping, traceroute, locating the failure hop)
- **H2.2** SQL Backend Server Error (VLAN and routing misconfiguration)
- **H2.3** 503 Service Unavailable (subnet mismatch)
- **H2.4** Traceroute Tool (filtering, suboptimal routing, ACLs, routing costs)
- **H2.5** TCP Server Connection (listeners, firewall rules, telnet, netstat)
- **H2.6** Application Port Testing
- **H2.7** Firewall Filtering Issues (stateful vs stateless, ACLs, DMZ, WAF)

### H3 Domain Name System (DNS)
- **H3.1** Introduction to DNS (UDP 53 with TCP fallback, DNS over HTTPS, DNS over TLS)
- **H3.2** How DNS Lookup Works (client cache, resolver, root, TLD, authoritative, TTL)
- **H3.3** Nslookup Command (cached vs authoritative answers; A, MX, NS)
- **H3.4** DNS Troubleshooting Workflow
- **H3.5** Linux Dig Command (A, AAAA, CNAME, MX, NS; tracing from root to authoritative)

### H4 Transport Layer Security (SSL/TLS)
- **H4.1** Introduction to TLS (in-transit encryption, CA-signed server authentication, integrity)
- **H4.2** TLS Handshake
- **H4.3** TLS 1.3: The Gold Standard (one-round-trip handshake, ALPN, session resumption, 0-RTT)
- **H4.4** SSL Certificate Validation (validity, expiry, issuer, revocation status, protocol support)
- **H4.5** OCSP Certificate Revocation (OCSP stapling)
- **H4.6** HTTP Strict Transport Security (HSTS)
- **H4.7** SSL Troubleshooting Workflow

### H5 Network Performance Deep Dive
- **H5.1** What is Network Latency? (propagation, transmission, processing, queuing, protocol delay)
- **H5.2** Troubleshooting Latency (MTR)
- **H5.3** TCP Protocol Delay (DNS, TCP and TLS handshakes; head-of-line blocking)
- **H5.4** Server Processing Delays (disk I/O)
- **H5.5** Bandwidth, Throughput, and Latency
- **H5.6** Packet Loss Dynamics
- **H5.7** System and Network Latency Comparison
- **H5.8** Low-Latency Testing (MTU and fragmentation)

### H6 Web Application Architecture
- **H6.1** How a Browser Connects to the Internet (DNS, ARP, TCP, TLS 1.3, HTTP)
- **H6.2** HTTP Request/Response Model
- **H6.3** Full Stack Application Flow
- **H6.4** Browser Caching (cache-control policies, ETag, proxy and CDN revalidation)
- **H6.5** Data Center Load Balancers (L7 routing, HTTPS termination, sticky sessions)
- **H6.6** Reverse Proxy Server Operation (TLS offloading, caching, compression)
- **H6.7** TCP Socket Buffers
- **H6.8** Application Troubleshooting Basics
- **H6.9** Time to First Byte (Server Delay)
- **H6.10** Application Chattiness (round trips, send window, RTT)
- **H6.11** Web Page Load Time (PLT)
- **H6.12** Content Delivery Network (CDN) (edge servers, IP anycast, BGP)
- **H6.13** HTTP/3 (QUIC)

---

## I. Still uncovered (◆, not from a course page)

Optional course for I1.2 and I1.3: "Ultimate DNS Warrior" on Udemy (split DNS, zone transfers, DNSSEC).

### I1 DNS
- **I1.1** Provider alias records vs CNAME (zone apex rule, health-aware aliases) ◆
- **I1.2** DNS routing policies: weighted, latency, geolocation, failover with health checks ◆
- **I1.3** Split-horizon DNS and DNSSEC ◆
- **I1.4** Directory-integrated DNS: domain controllers, conditional forwarders between cloud and on-prem ◆

### I2 TLS and Certificates
- **I2.1** Termination modes beyond offload: passthrough vs re-encryption to the backend ◆
- **I2.2** SNI and multiple certificates on one listener ◆
- **I2.3** Certificate lifecycle: wildcard vs SAN, ACME automation, renewal, expiry monitoring ◆
- **I2.4** Private CAs and internal trust chains ◆

### I3 Acceleration
- **I3.1** Anycast edge acceleration without caching vs CDN (static anycast IPs for allowlisting, non-HTTP traffic, fast regional failover) ◆

---

## Known gap

SLOs, error budgets, alerting, and incident response are not covered by any course above. The free Google SRE book is the suggested source for a future section J.

## Sources

- A: https://www.udemy.com/course/fundamentals-of-operating-systems/
- B: https://www.udemy.com/course/database-engines-crash-course/
- B10 to B14 listing: https://www.careers360.com/courses-certifications/udemy-fundamentals-of-database-engineering-course
- C: https://www.udemy.com/course/developer-to-architect/
- D: https://www.udemy.com/course/rocking-system-design/
- E: https://www.udemy.com/course/ai-system-design-for-engineers/
- F: https://www.udemy.com/course/fundamentals-of-networking-for-effective-backend-design/
- G: https://www.udemy.com/course/aws-certified-advanced-networking-specialty-ans/
- H: https://www.udemy.com/course/networking-fundamentals-for-full-stack-web-developers/
- I (optional DNS course): https://www.udemy.com/course/ultimate-dns-warrior/
