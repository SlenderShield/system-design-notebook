# Chapter 2: The Universal Design Checklist

> _"Give a person a design, they solve one problem. Give a person a checklist, they solve every problem."_

---

## 🎯 What You'll Learn

By the end of this chapter, you'll have:

- A **6-step repeatable autopilot** that works on ANY design problem (LLD or HLD)
- Each step with concrete sub-questions so you never get stuck
- Two fully worked examples — one LLD, one HLD — using the same checklist
- The ability to **structure your thinking** in interviews so you never freeze

---

## 2.1 Why a Checklist?

### The Problem: Blank Page Paralysis

You hear _"Design XYZ"_ and your brain goes:

```
🧠: "Um... should I start with the database? Or the API?
     Or the classes? Should I draw a diagram first? What kind
     of diagram? What if I pick the wrong starting point?"

⏱️: *40 minutes of time ticking away*
```

This happens to **everyone** — beginners AND experienced engineers. The problem isn't lack of knowledge. The problem is **no structured starting point**.

### The Solution: A Repeatable Pipeline

The 6-step pipeline from Chapter 1, now turned into a concrete checklist with sub-questions:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   1. CLARIFY  ──▶  2. ENTITIES  ──▶  3. FLOWS             │
│                                                             │
│   4. BOTTLENECKS  ──▶  5. PATTERNS  ──▶  6. TRADEOFFS     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**The rule**: Never skip a step. Never reorder them. Even if you think you already know the answer, walk through each step deliberately. The step you skip is the step that bites you.

---

## 2.2 The Checklist — Full Breakdown

### Step 1: CLARIFY Requirements

**Time in interview**: 3-5 minutes (DON'T skip this — it shows maturity)

**What you're doing**: Converting a vague problem into a concrete scope.

#### Sub-Questions to Ask:

**Functional (What?)**

- [ ] Who are the actors/users of this system?
- [ ] What are the **top 5** things they can do? (MVP only)
- [ ] What's explicitly **OUT of scope**?
- [ ] Are there different user roles? (Rider vs Driver, Viewer vs Uploader)

**Non-Functional (How well?)**

- [ ] Is this read-heavy or write-heavy? (ratio?)
- [ ] What's the expected scale? (users, requests/sec, data volume)
- [ ] Latency requirements? (real-time? near-real-time? batch is ok?)
- [ ] Availability requirements? (can it go down for maintenance?)
- [ ] Consistency requirements? (can data be stale for a few seconds?)

**Constraints (Boundaries)**

- [ ] Any technology constraints? (must use X, can't use Y)
- [ ] Regulatory constraints? (GDPR, PCI-DSS for payments)
- [ ] Budget/team constraints? (small team = simpler architecture)

#### The MVP Mindset

**Critical**: Your initial design should cover the **core flow only**. Not every edge case. Not every feature.

```
❌ "Let me design the entire system with admin panel, analytics,
    multi-language support, accessibility features..."

✅ "Let me focus on the core flow first: User places order →
    Restaurant receives order → Driver picks up → Delivery.
    Then we can discuss extensions."
```

The interviewer WANTS you to start simple. It shows engineering judgment.

---

### Step 2: IDENTIFY Entities / Core Objects

**Time in interview**: 3-5 minutes

**What you're doing**: Naming the core "things" in your system.

#### For LLD (Classes):

- [ ] What are the main **nouns** in the requirements? → these are candidate classes
- [ ] Which entities have **state** that changes? (mutable objects need careful design)
- [ ] Which entities just **hold data** vs which **perform actions**?
- [ ] Are there any **value objects** (immutable, compared by value)? (Money, Address, Location)

#### For HLD (Services + Data Stores):

- [ ] What are the main **data entities** that need storage?
- [ ] Which entities are queried together? (suggests they belong in the same store)
- [ ] What's the relationship between entities? (1:1, 1:N, N:N)
- [ ] How much data per entity? How fast does it grow?

#### The Entity Discovery Technique

Read your requirements and **underline the nouns**:

> _"A **user** can book a **ride**. The system assigns a **driver** with a **vehicle**.
> A **trip** is created with pickup and dropoff **locations**. After the trip, a **payment**
> is processed and both parties leave a **rating**."_

Your entities: `User`, `Ride`, `Driver`, `Vehicle`, `Trip`, `Location`, `Payment`, `Rating`

Not every noun becomes an entity — **Location** might just be an attribute of Trip, not its own class. That judgment comes with practice.

---

### Step 3: MAP Flows / Interactions

**Time in interview**: 5-7 minutes

**What you're doing**: Describing how entities interact to accomplish use cases.

#### Sub-Questions:

- [ ] What is the **happy path** for each core use case?
- [ ] What triggers each flow? (user action, timer, system event)
- [ ] What steps happen in sequence vs what can happen in parallel?
- [ ] Where does data flow FROM and TO?

#### For LLD — Think in method calls:

```
// Who calls whom?
Rider.requestRide(pickup, dropoff)
    → RideService.findNearbyDrivers(pickup)
        → DriverMatchingStrategy.selectBestDriver(drivers, pickup)
            → Driver.offerRide(rideRequest)
                → if accepted: Trip.create(rider, driver, route)
```

#### For HLD — Think in service interactions:

```
Rider App  ──▶  API Gateway  ──▶  Ride Service  ──▶  Matching Service
                                       │                    │
                                       ▼                    ▼
                                  Trip Service        Location Service
                                       │                    │
                                       ▼                    ▼
                                  Payment Service     Notification Service
```

#### The "Follow the Data" Technique

If you ever get stuck mapping flows, just follow the data:

> Where is data **created**? Where does it **move**? Where does it **end up**?

For a food delivery order:

1. **Created**: Rider selects items → Order object created in Order Service
2. **Moves**: Order sent to Restaurant Service → then to Driver Matching → then to Notification
3. **Ends up**: Order status in database, payment record, analytics event

---

### Step 4: FIND Bottlenecks

**Time in interview**: 5-7 minutes

**What you're doing**: Finding where the system will struggle, break, or slow down.

#### The Bottleneck Categories:

**For LLD:**

| Bottleneck Type    | Question                                               | Example                                                              |
| ------------------ | ------------------------------------------------------ | -------------------------------------------------------------------- |
| **Concurrency**    | Can two threads corrupt state?                         | Two users booking the same seat                                      |
| **God Object**     | Is one class doing too much?                           | A `RideManager` that handles matching, pricing, routing, AND payment |
| **Tight Coupling** | Does changing X force changing Y?                      | Changing payment logic requires modifying the Order class            |
| **Rigid Design**   | Can you add new types without modifying existing code? | Adding a new vehicle type requires if-else changes everywhere        |

**For HLD:**

| Bottleneck Type             | Question                                          | Example                                       |
| --------------------------- | ------------------------------------------------- | --------------------------------------------- |
| **Hot Spot**                | Is one component handling disproportionate load?  | All reads hitting a single DB                 |
| **Write Contention**        | Are many writers competing for the same resource? | Updating seat availability counter            |
| **Data Growth**             | What data grows unboundedly?                      | Chat messages, ride history, analytics events |
| **Fan-out**                 | Does one event trigger many downstream actions?   | One post → notify 1M followers                |
| **Single Point of Failure** | What happens if THIS component dies?              | Single database, single API server            |

#### The "What If" Game

Play this game with every component:

> _"What happens if this gets 100x more traffic?"_  
> _"What happens if this component crashes?"_  
> _"What happens if this data grows to 1 billion records?"_

The answers tell you exactly where to focus your design effort.

---

### Step 5: APPLY Patterns

**Time in interview**: 5-10 minutes

**What you're doing**: Using known solutions for the bottlenecks you found.

#### The Smell → Pattern Map (LLD):

| Smell / Bottleneck                         | Pattern               | Why                                           |
| ------------------------------------------ | --------------------- | --------------------------------------------- |
| Multiple algorithms / behaviors            | **Strategy**          | Swap algorithms without changing the caller   |
| Many event listeners need updates          | **Observer**          | Decouple event producer from consumers        |
| Complex object creation                    | **Builder / Factory** | Separate construction from representation     |
| Need to add behavior dynamically           | **Decorator**         | Wrap objects to add responsibilities          |
| Undo/redo, queued operations               | **Command**           | Encapsulate actions as objects                |
| Only one instance allowed                  | **Singleton**         | Global access point (use cautiously!)         |
| Need to iterate without exposing internals | **Iterator**          | Standard traversal without exposing structure |
| Object has distinct states + transitions   | **State**             | Replace conditionals with state objects       |

#### The Smell → Pattern Map (HLD):

| Smell / Bottleneck          | Pattern                             | Why                                      |
| --------------------------- | ----------------------------------- | ---------------------------------------- |
| Read-heavy traffic          | **Caching** (Redis, CDN)            | Reduce DB load by serving from memory    |
| Write-heavy traffic         | **Message Queue** (Kafka, RabbitMQ) | Buffer writes, process asynchronously    |
| Need global uniqueness      | **ID Generator** (Snowflake, UUID)  | Generate unique IDs without coordination |
| Single DB overloaded        | **Sharding**                        | Split data across multiple DBs           |
| Need to search text         | **Search Index** (Elasticsearch)    | Inverted index for fast text search      |
| Bursty traffic              | **Rate Limiter**                    | Protect services from overload           |
| Cross-service communication | **API Gateway**                     | Single entry point, routing, auth        |
| Need to track state changes | **Event Sourcing**                  | Store events, not just current state     |
| Complex workflows           | **Saga Pattern**                    | Manage distributed transactions          |

**Important**: Don't pattern-match blindly! Always explain **WHY** the pattern fits your specific bottleneck.

---

### Step 6: DEFEND Tradeoffs

**Time in interview**: Ongoing throughout

**What you're doing**: Explaining why you chose what you chose, and what the alternatives are.

#### The Tradeoff Defense Formula:

```
"I chose [X] because [specific reason tied to requirements].
 The alternative was [Y], but [specific reason Y doesn't fit].
 The tradeoff is [what I'm giving up], which is acceptable because [why it's ok]."
```

#### Example Tradeoff Defenses:

> **SQL vs NoSQL**: "I chose PostgreSQL because the data is relational (users have orders, orders have items) and we need ACID transactions for payments. I considered MongoDB, but the relational queries would require denormalization that creates update anomalies. The tradeoff is horizontal scaling is harder with SQL, but at our scale (100K users) a single instance with read replicas is sufficient."

> **Sync vs Async**: "I chose asynchronous processing via Kafka for notifications because sending push, email, and SMS synchronously would add 2-3 seconds to the API response time. The tradeoff is notifications arrive with a slight delay (1-5 seconds), which is perfectly acceptable — nobody expects instant SMS."

> **Cache**: "I added a Redis cache for user profiles because the same profile is read 100x more than it's updated (celebrity profiles on Instagram). I'll use cache-aside with a 5-minute TTL. The tradeoff is a user might see stale data for up to 5 minutes after a profile update, which is acceptable for this use case."

#### Questions Interviewers Love to Ask:

- _"Why not use [alternative technology]?"_
- _"What happens when [component X] fails?"_
- _"How would this change if the scale grows 100x?"_
- _"What would you do differently with more time / bigger team?"_

**If you don't have a good answer, be honest**: "That's a great question. I hadn't considered [X], but thinking about it now, I'd probably [Y] because [Z]." Honesty impresses more than BS.

---

## 2.3 Worked Example: LLD — Design a Movie Ticket Booking System

Let's walk through the entire checklist for an LLD problem.

### Step 1: CLARIFY

**Functional**:

- Users can browse movies showing at different theaters
- Users can view available seats for a show
- Users can book seats (1-10 per booking)
- Users can cancel a booking (with refund rules)
- System prevents double-booking of seats

**Non-Functional**:

- Handle concurrent seat bookings (the hardest part!)
- Low latency for seat availability check

**Out of scope**: Payment gateway integration, user authentication, recommendation engine

### Step 2: ENTITIES

```
Movie          – title, duration, genre, rating
Theater        – name, address, list of screens
Screen         – screenNumber, list of seats, theater
Seat           – row, number, type (REGULAR/PREMIUM/VIP)
Show           – movie, screen, startTime, endTime
ShowSeat       – show + seat + status (AVAILABLE/LOCKED/BOOKED)
Booking        – user, show, list of showSeats, status, totalAmount
User           – name, email
Payment        – booking, amount, status, method
```

**Key insight**: `ShowSeat` is a **separate entity** from `Seat`. Why? Because the same physical `Seat` (Row A, #5) exists across many shows. Its availability is **per show**, not per seat. This is a common modeling mistake.

### Step 3: FLOWS

**Happy Path — Book Tickets**:

```
User selects Movie
    → selects Show (date/time)
        → sees available seats (ShowSeat where status = AVAILABLE)
            → selects seats
                → system LOCKS seats temporarily (5-min hold)
                    → user proceeds to payment
                        → payment success → seats marked BOOKED
                        → payment fail / timeout → seats UNLOCKED
```

### Step 4: BOTTLENECKS

| Bottleneck             | Problem                                            | Severity          |
| ---------------------- | -------------------------------------------------- | ----------------- |
| **Concurrent booking** | Two users selecting the same seat at the same time | 🔴 Critical       |
| **Seat locking**       | Lock must expire if payment is abandoned           | 🟡 Important      |
| **Show browsing**      | Many users viewing same show's availability        | 🟢 Lower priority |

### Step 5: PATTERNS

| Bottleneck            | Pattern Applied             | Explanation                                                                         |
| --------------------- | --------------------------- | ----------------------------------------------------------------------------------- |
| Concurrent booking    | **Locking + State Pattern** | ShowSeat transitions: AVAILABLE → LOCKED → BOOKED. Only one thread can lock a seat. |
| Seat type pricing     | **Strategy**                | Different pricing for REGULAR vs PREMIUM vs VIP                                     |
| Booking notifications | **Observer**                | Notify email service, SMS service on booking confirmation                           |
| Seat lock expiry      | **Command + Scheduler**     | Schedule an "unlock" command to execute after 5 minutes                             |

### Step 6: TRADEOFFS

> "I use a **pessimistic lock** on seat selection rather than optimistic because with popular shows (Avengers opening night), conflict rate is very high. Optimistic locking would result in too many retries. The tradeoff is lower throughput on seat selection, but correctness matters more than speed here — double-booking a seat is unacceptable."

> "I separated `Seat` from `ShowSeat` because a seat is a physical entity that doesn't change, while its availability changes per show. This avoids duplicating seat data across thousands of shows."

---

## 2.4 Worked Example: HLD — Design a URL Shortener

Same checklist, HLD lens.

### Step 1: CLARIFY

**Functional**:

- Users submit a long URL → get a short URL back
- Short URL redirects to original long URL
- Optional: custom aliases, expiration, analytics (click count)

**Non-Functional**:

- Read-heavy: 1000:1 read-to-write ratio
- Low latency on redirect (< 50ms — users clicking links expect instant)
- 100M URLs created per month → need unique short codes at scale
- High availability: links must always resolve

**Out of scope**: User accounts (for MVP), link editing, spam detection

### Step 2: ENTITIES

```
URL Mapping:
    shortCode    – "abc123" (6-7 characters)
    longURL      – "https://very-long-url.com/path?query=..."
    createdAt    – timestamp
    expiresAt    – optional timestamp
    clickCount   – analytics counter

User (future scope):
    userId, email, apiKey
```

- Data model is **simple** — key-value nature (shortCode → longURL)
- No complex joins needed
- Data grows over time, never updated (write-once, read-many)

### Step 3: FLOWS

**Create Short URL**:

```
Client  ──▶  API Server  ──▶  ID Generator (create unique shortCode)
                                     │
                                     ▼
                              Store {shortCode → longURL} in DB
                                     │
                                     ▼
                              Return "https://short.ly/abc123"
```

**Redirect (the hot path)**:

```
Client clicks short.ly/abc123
    ──▶  API Server  ──▶  Check Cache (Redis)
                              │
                    ┌─────────┴──────────┐
                    │ HIT                │ MISS
                    ▼                    ▼
              Return longURL       Query DB → Store in cache
                    │                    │
                    ▼                    ▼
              HTTP 301/302 Redirect to longURL
```

### Step 4: BOTTLENECKS

| Bottleneck               | Problem                                                           | Severity     |
| ------------------------ | ----------------------------------------------------------------- | ------------ |
| **Read volume**          | 1000:1 ratio, billions of redirects/month                         | 🔴 Critical  |
| **Unique ID generation** | Must generate globally unique short codes at scale, no collisions | 🔴 Critical  |
| **Storage growth**       | 100M URLs/month × 1KB = 100GB/month linear growth                 | 🟡 Important |
| **Hot URLs**             | Some URLs get millions of clicks (viral content)                  | 🟡 Important |

### Step 5: PATTERNS

| Bottleneck     | Pattern                    | Solution                                                                               |
| -------------- | -------------------------- | -------------------------------------------------------------------------------------- |
| Read volume    | **Caching**                | Redis cache-aside with TTL for URL mappings                                            |
| Unique IDs     | **Pre-generated ID Range** | Each server pre-fetches a range of IDs from a counter service, no runtime coordination |
| Storage growth | **NoSQL / Sharding**       | DynamoDB or Cassandra — key-value model, horizontally scalable                         |
| Hot URLs       | **CDN + Cache Tiers**      | Most popular URLs served from edge cache                                               |

### Step 6: TRADEOFFS

> "I chose **NoSQL (DynamoDB)** over SQL because: (a) the data model is pure key-value, (b) I don't need joins or transactions, (c) I need horizontal scaling as data grows. Tradeoff: no relational queries, but I don't need any."

> "For unique IDs, I chose **pre-generated ranges** over hashing the URL (MD5/SHA) because hashing can produce collisions that require checking the DB. Pre-generated sequential IDs are guaranteed unique. Tradeoff: requires a centralized counter service, but a single counter with range allocation handles billions of IDs."

> "I use **HTTP 301 (permanent redirect)** for non-expiring URLs so browsers cache the redirect and don't hit my server again. For URLs with analytics (click counting), I use **302 (temporary redirect)** to force every click through my server. Tradeoff: 301 reduces load but loses analytics visibility."

---

## 2.5 LLD vs HLD Checklist — Side by Side

| Step               | LLD Question                              | HLD Question                                    |
| ------------------ | ----------------------------------------- | ----------------------------------------------- |
| **1. Clarify**     | What use cases? What rules?               | What scale? What latency? Read/write ratio?     |
| **2. Entities**    | What classes? What attributes?            | What services? What data stores?                |
| **3. Flows**       | What method calls? What returns what?     | What API calls? What service talks to what?     |
| **4. Bottlenecks** | Concurrency? God objects? Tight coupling? | Hot spots? Fan-out? Data growth? SPOF?          |
| **5. Patterns**    | Strategy, Observer, Factory, State...     | Cache, Queue, Shard, Saga, Rate Limiter...      |
| **6. Tradeoffs**   | Why this pattern over that one?           | Why this technology/architecture over that one? |

---

## 2.6 The Checklist as Interview Autopilot

Here's how this maps to a 45-minute system design interview:

```
┌────────────────────────────────────────────────────────────────┐
│  0:00 - 0:05  │  CLARIFY: Ask questions, define scope          │
│  0:05 - 0:10  │  ENTITIES: Core objects / services / data       │
│  0:10 - 0:20  │  FLOWS: Main workflows, API design              │
│  0:20 - 0:30  │  BOTTLENECKS → PATTERNS: Find & solve           │
│  0:30 - 0:40  │  DEEP DIVE: Interviewer picks area to explore   │
│  0:40 - 0:45  │  TRADEOFFS: Defend + discuss evolution          │
└────────────────────────────────────────────────────────────────┘
```

**Key**: The interviewer will usually want to deep-dive into one area (Step 5 on the diagram). The checklist ensures you reach that point efficiently instead of wandering.

---

## 🧪 Exercises

### Exercise 1: Checklist Speedrun — LLD

Apply the 6-step checklist to: **"Design a Library Management System"**

Don't spend more than 15 minutes. Write:

1. 3 functional requirements (MVP only)
2. Your entities (list of nouns)
3. One happy-path flow (book borrowing)
4. The #1 bottleneck
5. One pattern you'd apply
6. One tradeoff you'd defend

### Exercise 2: Checklist Speedrun — HLD

Apply the 6-step checklist to: **"Design a Paste Bin (like pastebin.com)"**

Same format:

1. 3 functional requirements
2. Core entities / data model
3. The read + write flows
4. The #1 bottleneck
5. One pattern you'd apply
6. One tradeoff you'd defend

### Exercise 3: Smell Detector

For each scenario, identify the bottleneck TYPE and the pattern you'd apply:

| Scenario                                                          | Bottleneck Type? | Pattern? |
| ----------------------------------------------------------------- | ---------------- | -------- |
| A social media post goes viral, 10M people view it                | ?                | ?        |
| Two users try to book the last movie ticket at the same time      | ?                | ?        |
| A notification needs to be sent via email, SMS, AND push          | ?                | ?        |
| Pricing rules change every quarter (flat rate, per-minute, surge) | ?                | ?        |
| A micro-service calls another service but it's down               | ?                | ?        |

---

## 📝 Chapter Summary

| Concept              | Key Takeaway                                                                                               |
| -------------------- | ---------------------------------------------------------------------------------------------------------- |
| The 6-Step Checklist | Clarify → Entities → Flows → Bottlenecks → Patterns → Tradeoffs                                            |
| Step 1 (Clarify)     | MVP first. Ask about scale, read/write ratio, latency, consistency                                         |
| Step 2 (Entities)    | Underline the nouns. Separate data holders from action performers                                          |
| Step 3 (Flows)       | Follow the data. LLD = method calls. HLD = service interactions                                            |
| Step 4 (Bottlenecks) | Play "What if 100x?" and "What if it crashes?"                                                             |
| Step 5 (Patterns)    | Smell → Pattern mapping. Never apply without explaining WHY                                                |
| Step 6 (Tradeoffs)   | "I chose X because Y. The alternative was Z. The tradeoff is W."                                           |
| Interview Time       | 5 min clarify, 5 min entities, 10 min flows, 10 min bottleneck+patterns, 10 min deep-dive, 5 min tradeoffs |

---

## ⏭️ What's Next

**Chapter 3: Pattern Recognition System (Design Edition)** — We'll build a complete "smell → pattern" recognition system. You'll learn to instantly recognize what pattern a problem needs, just by reading the requirements. This is the design equivalent of "seeing the matrix."

---

> _"A checklist doesn't make you robotic. It frees your brain to focus on the creative parts — the parts that actually matter."_
