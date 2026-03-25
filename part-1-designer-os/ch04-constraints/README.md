# Chapter 4: Constraints = Secret Answer Key

> _"Don't fight constraints. Read them. They're telling you exactly what to build."_

---

## 🎯 What You'll Learn

By the end of this chapter, you'll:

- See non-functional requirements as **design instructions**, not vague wishes
- Master the **Constraint → Architecture** decoder ring
- Know how to extract hidden constraints the interviewer didn't state
- Use back-of-the-envelope math to **quantify** constraints
- Never say "it should be scalable" again — you'll say exactly HOW scalable and WHY it matters

---

## 4.1 The Mindset Shift

### How Beginners See Constraints

```
Interviewer: "The system needs to handle 10 million daily active users."

Beginner's brain: 😰 "That's a lot... I guess I need to make it scalable?"
```

Vague. Useless. Doesn't drive any decision.

### How Experts See Constraints

```
Interviewer: "The system needs to handle 10 million daily active users."

Expert's brain:
  → 10M DAU
  → Assume 10% concurrency at peak = 1M concurrent users
  → Each user makes ~10 requests/day = 100M requests/day
  → 100M / 86400 seconds = ~1150 requests/sec average
  → Peak = 3-5x average = ~5000 req/sec
  → ONE server handles ~500-1000 req/sec
  → Need: 5-10 servers + load balancer
  → This is NOT "massive scale" — no sharding needed yet
  → Simple horizontal scaling with read replicas handles this
```

**Same constraint. Completely different signal.** The expert turned a scary number into a concrete architecture decision in 30 seconds.

---

## 4.2 The Constraint → Architecture Decoder

Every non-functional constraint maps to specific architectural components. This is the decoder ring:

### 🔴 Scale Constraints

| Constraint                     | What It Tells You          | Architecture Response                           |
| ------------------------------ | -------------------------- | ----------------------------------------------- |
| **High DAU (>1M)**             | Multiple servers needed    | Load balancer + horizontal scaling              |
| **Very high DAU (>100M)**      | Single region won't cut it | Multi-region deployment + geo-routing           |
| **High concurrent writes**     | Single DB bottleneck       | Write-ahead log, sharding, async writes         |
| **High concurrent reads**      | DB overloaded by reads     | Read replicas + caching layer                   |
| **Massive data volume (>1TB)** | Won't fit in one DB        | Sharding + archival strategy                    |
| **Rapid data growth**          | Storage costs explode      | TTL policies, cold storage tiering, compression |

### 🟡 Latency Constraints

| Constraint                   | What It Tells You                     | Architecture Response                           |
| ---------------------------- | ------------------------------------- | ----------------------------------------------- |
| **< 50ms response**          | Can't afford DB round-trip every time | In-memory cache (Redis) is PRIMARY read path    |
| **< 200ms response**         | Cache miss can hit DB, but optimize   | Cache + DB with connection pooling              |
| **< 1 second**               | Comfortable — standard web app        | Standard architecture, no special optimizations |
| **Real-time updates**        | HTTP polling too slow                 | WebSocket or Server-Sent Events (SSE)           |
| **Near-real-time (< 5 sec)** | Small delay acceptable                | Pub/Sub with streaming consumers                |

### 🟢 Availability Constraints

| Constraint                        | What It Tells You                  | Architecture Response                             |
| --------------------------------- | ---------------------------------- | ------------------------------------------------- |
| **99.9% (8.7 hrs downtime/year)** | Standard web app                   | Multiple servers, health checks, auto-restart     |
| **99.99% (52 min downtime/year)** | No single points of failure        | Redundancy at every layer, failover, multi-AZ     |
| **99.999% (5 min downtime/year)** | Mission-critical (payment, health) | Multi-region active-active, zero-downtime deploys |

### 🔵 Consistency Constraints

| Constraint                      | What It Tells You                              | Architecture Response                                 |
| ------------------------------- | ---------------------------------------------- | ----------------------------------------------------- |
| **Strong consistency required** | "Everyone sees the same data at the same time" | Single-leader replication, synchronous writes         |
| **Eventual consistency ok**     | "A few seconds stale is fine"                  | Multi-leader, async replication, caching with TTL     |
| **Consistency for SOME data**   | "Balance is always correct, feed can be stale" | Hybrid: strong for payments, eventual for social feed |

**The key insight**: Most systems need **different consistency levels for different data**. Payment balances need strong consistency. Social media feeds can be eventually consistent. This hybrid approach is what real systems use.

---

## 4.3 Back-of-the-Envelope Math (Your Secret Weapon)

This is the skill that makes interviewers say _"this person gets it."_ Let me teach you the mental math toolkit.

### The Numbers You Must Memorize

```
╔════════════════════════════════════════════════════════════╗
║  CAPACITY REFERENCE CARD                                  ║
╠════════════════════════════════════════════════════════════╣
║                                                            ║
║  TIME                                                      ║
║  1 day    = 86,400 seconds  (round to ~100K for math)     ║
║  1 month  = 2.5 million seconds (round to ~2.5M)          ║
║                                                            ║
║  STORAGE                                                   ║
║  1 char   = 1 byte (ASCII) / 2 bytes (Unicode)            ║
║  1 tweet  = ~300 bytes (text + metadata)                   ║
║  1 URL    = ~500 bytes (short code + long URL + metadata)  ║
║  1 photo  = ~200 KB (compressed)                           ║
║  1 video  = ~50 MB/minute (compressed, 720p)              ║
║  1 user profile = ~1 KB                                    ║
║                                                            ║
║  THROUGHPUT                                                ║
║  1 web server        = 500-1000 req/sec (typical)         ║
║  1 SQL DB (single)   = 5,000-10,000 queries/sec           ║
║  1 Redis instance    = 100,000 ops/sec                    ║
║  1 Kafka broker      = 100,000 messages/sec               ║
║                                                            ║
║  NETWORK                                                   ║
║  Same datacenter RTT   = 0.5 ms                           ║
║  Cross-datacenter RTT  = 50-100 ms                        ║
║  Cross-continent RTT   = 150-300 ms                       ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝
```

### The Estimation Framework

**3-step mental math** for any scale question:

```
Step 1: TOTAL VOLUME     = users × actions × data_per_action
Step 2: RATE (per second) = total_volume / seconds_in_period
Step 3: PEAK RATE         = average_rate × peak_multiplier (3-5x)
```

### Worked Example: Instagram-like Photo Sharing

> _"50M DAU, each uploads 2 photos/day, each views 100 photos/day."_

**Storage estimation**:

```
Writes:  50M users × 2 photos × 200 KB = 20 TB/day
         20 TB × 365 = 7.3 PB/year    → need distributed storage + CDN
```

**Throughput estimation**:

```
Writes:  50M × 2 = 100M uploads/day
         100M / 100K sec = 1,000 writes/sec → manageable for distributed DB

Reads:   50M × 100 = 5 billion views/day
         5B / 100K sec = 50,000 reads/sec → need heavy caching + CDN

Read:Write ratio = 50,000 : 1,000 = 50:1 → READ-HEAVY → CACHE IS CRITICAL
```

**What the math told us** (without guessing):

- ✅ Storage: Distributed storage (S3/HDFS) + CDN, not a single DB
- ✅ Reads: Cache layer is mandatory (50:1 ratio)
- ✅ Writes: 1K/sec is manageable — don't over-engineer the write path
- ✅ Growth: 7.3 PB/year means you need a data retention/archival strategy

**See how the numbers eliminated guesswork?** You didn't "decide" to use a cache — the math **proved** you need one.

---

## 4.4 Hidden Constraints (The Ones Nobody States)

Some of the most important constraints are never explicitly stated. Great designers **infer** them.

### Hidden Constraint 1: Read:Write Ratio

Almost never stated directly, but it **determines your entire data architecture**.

| System             | Approximate Ratio                    | Implication                          |
| ------------------ | ------------------------------------ | ------------------------------------ |
| Social media feed  | 1000:1 (read-heavy)                  | Heavy caching, pre-computed feeds    |
| Chat messaging     | 1:1 (balanced)                       | Optimize both paths equally          |
| Logging/analytics  | 1:100 (write-heavy)                  | Append-only storage, batch reads     |
| E-commerce catalog | 100:1 (read-heavy)                   | Cache, CDN, denormalize for reads    |
| Stock trading      | 1:10 (write-heavy, latency-critical) | In-memory processing, event sourcing |

**How to extract it**: _"For every write, how many reads will happen on that data?"_

### Hidden Constraint 2: Data Access Patterns

| Pattern                                   | Implication                                 |
| ----------------------------------------- | ------------------------------------------- |
| **Temporal**: mostly recent data accessed | Time-based partitioning, archive old data   |
| **Spatial**: by location                  | Geo-sharding, CDN with edge nodes           |
| **Social**: by relationships              | Graph database or adjacency-list modeling   |
| **Hot/Cold**: some data accessed way more | Tiered storage, hot cache for popular items |

**How to extract it**: _"Which 20% of the data handles 80% of the traffic?"_

### Hidden Constraint 3: Failure Domain

| Question                                    | Implication                                           |
| ------------------------------------------- | ----------------------------------------------------- |
| "What if a server crashes mid-transaction?" | Need transaction rollback, WAL, saga pattern          |
| "What if the entire datacenter goes down?"  | Need multi-AZ or multi-region                         |
| "What if a third-party API is down?"        | Need circuit breaker, fallback, retry queue           |
| "What if a deploy goes wrong?"              | Need blue-green deploy, canary release, feature flags |

**How to extract it**: _"What's the worst thing that can happen, and what does it cost the business?"_

### Hidden Constraint 4: Consistency Boundaries

This is the most subtle one. Ask yourself:

> _"Where does this system absolutely NEED strong consistency, and where can it tolerate staleness?"_

| System         | Must Be Consistent                                     | Can Be Eventually Consistent              |
| -------------- | ------------------------------------------------------ | ----------------------------------------- |
| **Uber**       | Driver-rider matching (can't assign same driver twice) | ETA display (a few seconds stale is fine) |
| **Instagram**  | Like count eventually                                  | Feed content (seconds-stale is fine)      |
| **Banking**    | Account balance, transfers                             | Transaction history display               |
| **E-commerce** | Inventory during checkout                              | Product reviews                           |

**Why this matters**: Engineers who say "everything must be consistent" build systems that are slow, expensive, and fragile. Engineers who understand consistency boundaries build systems that are fast WHERE SPEED MATTERS and correct WHERE CORRECTNESS MATTERS.

---

## 4.5 The Constraint-Driven Design Walkthrough

Let's do a complete example where constraints literally dictate every architectural decision.

### Problem: Design a Real-Time Chat System (like WhatsApp)

#### Step 1: Extract Constraints

**Stated by interviewer**:

- 500M daily active users
- Messages must be delivered in < 1 second
- Messages must never be lost
- Support 1-to-1 and group chat (max 256 members)

**Inferred by you** (this is where you shine):

| Hidden Constraint | How You Found It                                      | Value                              |
| ----------------- | ----------------------------------------------------- | ---------------------------------- |
| Read:Write ratio  | Chat is roughly balanced — every sent message is read | ~1:1 for 1-on-1, 1:N for groups    |
| Data volume       | 500M users × 40 messages/day × 100 bytes = 2TB/day    | Massive write volume               |
| Connection model  | < 1s delivery → can't use polling                     | Persistent connections (WebSocket) |
| Ordering          | Messages in a conversation must appear in order       | Per-conversation ordering needed   |
| Offline delivery  | Users aren't always online                            | Message queue / store-and-forward  |

#### Step 2: Let Constraints Drive Architecture

| Constraint                           | →   | Architecture Decision                                                                 |
| ------------------------------------ | --- | ------------------------------------------------------------------------------------- |
| 500M DAU with persistent connections | →   | Millions of concurrent WebSocket connections → horizontally scaled connection servers |
| < 1 second delivery                  | →   | Can't route through a central DB for every message → in-memory message routing        |
| Messages must never be lost          | →   | Write-ahead log before delivery → persistent message store → confirmation protocol    |
| 2TB/day of messages                  | →   | Single DB impossible → shard by conversation_id or user_id                            |
| Per-conversation ordering            | →   | Messages within a conversation go to SAME partition → ordered within partition        |
| Offline users                        | →   | Store messages in "pending queue" per user → deliver when they reconnect              |
| Group chat (256 members)             | →   | Fan-out on write: store once for group → deliver to each member's connection          |

**Notice**: Every single architecture decision was driven by a specific constraint. Nothing was added "just because." No guessing. No "let's add Redis for fun."

#### Step 3: The Architecture That Emerges

```
                        ┌──────────────┐
    User Devices ──────▶│ Load Balancer │
    (WebSocket)         └──────┬───────┘
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
              ┌──────────┐┌──────────┐┌──────────┐
              │ Chat     ││ Chat     ││ Chat     │  ← Horizontally scaled
              │ Server 1 ││ Server 2 ││ Server N │    (WebSocket handlers)
              └────┬─────┘└────┬─────┘└────┬─────┘
                   │           │           │
                   └─────────┬─┘───────────┘
                             ▼
                   ┌─────────────────┐
                   │ Message Router   │  ← Routes by conversation_id
                   │ (in-memory)      │    to correct chat server
                   └────────┬────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Message  │  │ Message  │  │ Message  │  ← Sharded by
        │ Store    │  │ Store    │  │ Store    │    conversation_id
        │ Shard 1  │  │ Shard 2  │  │ Shard N  │
        └──────────┘  └──────────┘  └──────────┘

              ┌──────────────────────┐
              │ Pending Message Queue │  ← For offline users
              │ (per user)            │
              └──────────────────────┘
```

**Every box exists because a constraint demanded it.** Remove any constraint and the corresponding box becomes unnecessary.

---

## 4.6 The Constraint Extraction Checklist

Use this in every design interview. Ask these questions even if the interviewer doesn't bring them up:

```
┌────────────────────────────────────────────────────────────┐
│                CONSTRAINT EXTRACTION CHECKLIST              │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  SCALE                                                     │
│  □ How many total users?                                   │
│  □ How many daily active users (DAU)?                      │
│  □ How many requests per second at peak?                   │
│  □ How much data is generated per day?                     │
│                                                            │
│  LATENCY                                                   │
│  □ What response time is acceptable?                       │
│  □ Does the user need real-time updates?                   │
│  □ Is near-real-time (seconds) acceptable or instant?      │
│                                                            │
│  AVAILABILITY                                              │
│  □ What uptime is expected? (99.9%? 99.99%?)              │
│  □ Can there be planned maintenance windows?               │
│  □ What's the cost of 1 hour of downtime?                 │
│                                                            │
│  CONSISTENCY                                               │
│  □ Which data MUST be strongly consistent?                 │
│  □ Which data can tolerate staleness? For how long?        │
│  □ Are there financial transactions? (→ ACID)              │
│                                                            │
│  ACCESS PATTERNS                                           │
│  □ Read-heavy or write-heavy?                             │
│  □ Which data is "hot"? (accessed frequently)             │
│  □ Is access pattern temporal, spatial, or social?         │
│                                                            │
│  GROWTH                                                    │
│  □ How fast is data growing?                              │
│  □ Will user base grow 10x in the next year?              │
│  □ Which features are planned for v2?                      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 4.7 Constraints at the LLD Level

Constraints aren't just an HLD thing. In LLD, constraints drive patterns too:

| LLD Constraint                                            | Design Decision                                   |
| --------------------------------------------------------- | ------------------------------------------------- |
| "Must support new vehicle types without code changes"     | → Open/Closed Principle → Strategy/Factory        |
| "Multiple threads access the booking system"              | → Thread-safe design → Locking / Synchronized     |
| "Actions must be undoable"                                | → Command pattern with history stack              |
| "Object creation is complex with many optional fields"    | → Builder pattern                                 |
| "Behavior varies by state, transitions must be validated" | → State pattern                                   |
| "System must be testable in isolation"                    | → Dependency Injection, programming to interfaces |

**The same principle applies**: The constraint tells you the pattern. You don't invent solutions — you decode constraints.

---

## 🧪 Exercises

### Exercise 1: Constraint → Architecture Mapping

For each constraint, write the architectural response:

| #   | Constraint                                           | Your Architecture Response |
| --- | ---------------------------------------------------- | -------------------------- |
| 1   | "200M users, 90% of traffic is read requests"        | ?                          |
| 2   | "Payment data — zero tolerance for inconsistency"    | ?                          |
| 3   | "Global users across 6 continents, < 100ms response" | ?                          |
| 4   | "Users upload 50M images per day, each ~200KB"       | ?                          |
| 5   | "Feed should update within 5 seconds of a new post"  | ?                          |

### Exercise 2: Back-of-the-Envelope

A Twitter-like system with:

- 300M DAU
- Average user tweets 2x/day, reads 200 tweets/day
- Each tweet = 300 bytes (text + metadata)

Calculate:

1. Total tweets per day
2. Tweets per second (average and peak)
3. Read requests per second (average and peak)
4. Read:Write ratio
5. Daily storage for tweets only
6. Based on your numbers — what's the #1 architectural priority?

### Exercise 3: Hidden Constraint Extraction

You're asked: _"Design a ride-sharing system like Uber."_

The interviewer says only: _"It should work for a city with 1 million users and 50,000 drivers."_

List **5 hidden constraints** the interviewer didn't state, and for each, explain what architectural decision it drives.

---

## 📝 Chapter Summary

| Concept                         | Key Takeaway                                                               |
| ------------------------------- | -------------------------------------------------------------------------- |
| Constraints are answers         | Every NFR maps to a specific architecture component                        |
| Back-of-envelope math           | Turns scary numbers into concrete decisions in 30 seconds                  |
| Hidden constraints              | Read:write ratio, access patterns, failure domains, consistency boundaries |
| Constraint extraction checklist | Scale, Latency, Availability, Consistency, Access Patterns, Growth         |
| Constraint-driven design        | Every component exists because a constraint demanded it                    |
| LLD constraints too             | Thread safety, extensibility, testability → drive pattern choices          |

---

## ⏭️ What's Next

**Chapter 5: Tradeoffs & Engineering Judgment** — The final piece of the Designer OS. You'll learn to argue both sides of any design decision, understand when "it depends" is the correct answer, and develop the engineering judgment that separates good engineers from great architects.

---

> _"The best architects don't invent solutions. They let the constraints reveal the architecture."_
