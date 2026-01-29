# 🛡️ Robust Systems: Transactions & Locking Guide

A beginner-friendly guide to designing systems that handle data correctly under high concurrency (e.g., Flash Sales, Banking, Ticket Booking).

> **Core Philosophy:** Designing a robust system is like managing a busy intersection. Without traffic lights (locks) and rules of the road (transactions), you get crashes (corruption) and gridlock (deadlocks).

---

## 1. The Transaction (ACID)

**Definition:** A unit of work that treats multiple steps as a single "all-or-nothing" operation.

### The ACID Properties

To be robust, a transaction must pass the **ACID** test.

- **⚛️ A - Atomicity (The "Nuclear" Option)**
  - **Concept:** All steps happen, or none happen. No "half-done" work.
  - **Analogy:** If payment fails, you don't get "half a seat."
- **⚖️ C - Consistency (The Rule Book)**
  - **Concept:** Data must move from one valid state to another.
  - **Analogy:** Inventory cannot go below zero. The math must balance.
- **🙈 I - Isolation (The "Blinders")**
  - **Concept:** Concurrent transactions shouldn't see each other's messy intermediate work.
  - **Analogy:** While booking, seat 1A shouldn't look "Free" or "Taken" to others—it's invisible until finished.
- **💾 D - Durability (The "Stone Tablet")**
  - **Concept:** Once committed, data is saved permanently, even if the server crashes 1ms later.

---

## 2. Locking Mechanisms

How we manage access to shared resources.

### A. Pessimistic Locking ("Trust No One")

- **Mechanism:** Lock the resource **before** you touch it.
- **SQL:** `SELECT ... FOR UPDATE`
- **Analogy:** Taking the key to the fitting room. No one else can enter until you leave.
- **Best For:** High contention, high-risk data (e.g., Bank Transfers).

### B. Optimistic Locking ("Trust but Verify")

- **Mechanism:** Read freely, but check for changes **at the very end** (usually via a Version Number).
- **Analogy:** Editing a shared doc offline. If someone else edited it while you were offline, your upload is rejected ("Merge Conflict").
- **Best For:** Low contention, user profiles.

---

## 3. Database Isolation Levels

_Question: "How much of the messy, unfinished work of other users can I see?"_

| Level                           | Analogy                                                                                                           | Pros / Cons                           |
| :------------------------------ | :---------------------------------------------------------------------------------------------------------------- | :------------------------------------ |
| **1. Read Uncommitted**         | **The Peeping Tom:** You see data before it is saved (Dirty Reads).                                               | 🚀 Fast but Dangerous.                |
| **2. Read Committed** (Default) | **The Official Feed:** You only see data that has been officially Saved/Committed.                                | ⚡ Good balance. (Industry Standard). |
| **3. Repeatable Read**          | **The Photograph:** You look at a snapshot taken when you started. New changes (Phantoms) don't affect your view. | 🐢 Consistent but data might be old.  |
| **4. Serializable**             | **The Turnstile:** One person enters at a time.                                                                   | 🐌 Perfect accuracy, slowest speed.   |

---

## 4. The "Ticketmaster" Problem (Concurrency)

**Scenario:** 10,000 users try to buy "Seat 1A" at the exact same second.

### Solution 1: The Atomic Update (Preferred)

We avoid heavy explicit locking by using the database's internal **Write Lock** (Row Lock), which lasts only microseconds.

```sql
UPDATE seats
SET status = 'HELD', user_id = 'Alice'
WHERE id = '1A' AND status = 'AVAILABLE'; -- The Guard Clause
```

## Atomic Update Outcomes

- **Alice:** `1 row updated` → Success.
- **Bob:** `0 rows updated` → Fails immediately (**Fail Fast**).

---

## Solution 2: The Two-Step Gap

If you MUST **Read first** and then **Write**, you must bridge the gap — otherwise a **Race Condition** will occur.

- **Wrong:**  
  `SELECT ...` → _(gap)_ → `UPDATE`

- **Right:**  
  `SELECT ... FOR UPDATE` → _(Bob is blocked)_ → `UPDATE`

---

## 5. Deadlocks

### The Scenario: A Deadly Embrace

- Alice holds **Lock A**, waits for **Lock B**.
- Bob holds **Lock B**, waits for **Lock A**.

**The DB Response:**  
The "Hall Monitor" detects the cycle and **kills** one transaction (Rollback) to clear the jam.

### The Fix: Canonical Ordering

**Rule:** Always acquire locks in a globally consistent order (e.g., Low ID → High ID).

```python
# Sort requests before touching the DB
ids_to_lock = sorted(user_request_ids)

# Now everyone enters the "bridge" from the same side.
# A cycle is mathematically impossible.
```

## 6. Full Stack Strategy

### Database Layer

- Uses **Row Locking** (Surgical) instead of **Table Locking** (Nuclear).
- Uses **Atomic Updates** for speed.

### Backend Layer

- Catches `"0 rows updated"` → Returns **409 Conflict**.
- Catches `"Deadlock Error"` → Retries logic.
- Uses **Exponential Backoff** (wait 1s, 2s, 4s...) to prevent server spamming.

### Frontend Layer

- **No Limbo:** A seat is either _Available_ or _Taken_. No _pending_ state visible to others.
- **WebSockets:** Push updates (`"Seat 1A Sold!"`) instantly to turn seats grey, preventing **Ghost Clicks**.

---

## Cheat Sheet

| Strategy                | Speed     | Safety    | Best Use Case                               |
| ----------------------- | --------- | --------- | ------------------------------------------- |
| **Atomic Update**       | 🚀 High   | ✅ High   | Simple status changes (Tickets, Inventory). |
| **Select...For Update** | 🐢 Low    | ✅ High   | Complex financial logic requiring checks.   |
| **Optimistic Lock**     | ⚡ Medium | ⚠️ Medium | Long forms, Wikis, User Profiles.           |
