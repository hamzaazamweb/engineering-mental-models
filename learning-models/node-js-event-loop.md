# Node.js Event Loop & Architecture Notes

## Page 1: Order of Operations

### Hierarchy of Execution

1. **Call Stack:** Current synchronous code running.
2. **NextTick Queue:** Created by `process.nextTick`.
3. **Microtask Queue:** Created by Promises (queues microtasks).
4. **The Event Loop:** Includes Timers, I/O, `setImmediate` (Check phase).

### Event Loop Order of Operations

The loop flows in this specific order:
↓ **Timers**
↓ **Pending Callbacks**
↓ **Idle, Prepare**
↓ **Poll** (I/O)
↓ **Check**
↓ **Close Callbacks** s

---

## Page 2: process.nextTick() Deep Dive

**Mechanism:** `process.nextTick()` → Uses `nextTickQueue`

> **Definition:** Anything passed to it will run after the current operation finishes.

- **Joke:** The name is a lie (it doesn't wait for the next "tick").
- **Superpower:** It cuts the line (VIP status).

### Three Main Jobs

1. **Fixes Timing.**
2. **Runs after stack is clear.**
3. **Error Handling/Setup.**

### The Danger

- **Starvation:** It can starve the waiter (single thread/main).
- **The Risk:** If the waiter keeps serving VIPs, no one else will get a chance.

### Purpose

Helps Node.js ensure it finishes setup (event listeners and code setup) before the actual listening begins (on event).

---

## Page 3: The Waiter Analogy

### Node.js Behavior

- **The Waiter:** Main JS Single Thread.
- **The Routine:** He follows a strict routine called the Event Loop.
- **Work Flow:** When his hands are empty (**Stack Empty**), he checks the counters (queues in each phase) to see if there is any work to deliver.

---

## Page 4: I/O vs. CPU

### Comparison

- **Node is:** Super fast at waiting (**I/O**).
- **Node is:** Very slow at thinking (**CPU**) — e.g., Resizing a huge image.

### The Trap

If a callback has heavy math:

1. It is picked up by the main thread (single cycle).
2. The waiter cannot process other callbacks or accept new work into his hands.
3. He is blocked until he finishes the computation of the current task on the stack.

### The Fix

We fix this problem by using **Worker Threads**.

---

## Page 5: Summary & Analogies

### Architecture Entities

- **Event Loop Phases:** Managed by `libuv`.
- **NextTick / Microtask Queue:** Managed by Node.js itself.

### Key Analogies & Definitions

| Concept                | Analogy / Definition                    |
| :--------------------- | :-------------------------------------- |
| **Pending Callback**   | Returned mail bin                       |
| **Close Callback**     | The Janitor                             |
| **process.nextTick()** | VIP Pass                                |
| **setImmediate()**     | Runs inside the Check Phase (after I/O) |
| **Main JS Thread**     | The Waiter                              |

# The Event Loop in Action: An Express.js Example

This document illustrates how the Node.js Event Loop handles different types of requests in a real-world Express application. It demonstrates synchronous execution, asynchronous I/O delegation, and the dangers of blocking the loop.

## 1. The Code (The Menu)

Imagine this code is running a restaurant. It has three distinct "buttons" (routes) a user can press.

```javascript
const express = require("express");
const fs = require("fs"); // File System module (I/O operations)
const app = express();

// --- ROUTE 1: The "Fast Food" (Synchronous) ---
app.get("/", (req, res) => {
  console.log("A. Fast Route Hit");
  res.send("Hello World!");
});

// --- ROUTE 2: The "Slow Cooked" (Asynchronous I/O) ---
app.get("/read-file", (req, res) => {
  console.log("B. Reading File Started");

  // The Waiter delegates this to the Kitchen (Kernel/Thread Pool)
  fs.readFile("./big-file.txt", "utf8", (err, data) => {
    // This callback runs LATER in the Poll Phase
    console.log("C. File Read Complete");
    res.send("File content sent!");
  });

  console.log("D. Reading File Delegated");
});

// --- ROUTE 3: The "Bad Chef" (CPU Blocking) ---
app.get("/block-me", (req, res) => {
  console.log("E. Blocking Route Hit");

  // HEAVY MATH - The Waiter is trapped here!
  // This blocks the Single Thread.
  let count = 0;
  while (count < 5000000000) {
    count++;
  }

  console.log("F. Blocking Finished");
  res.send("Done!");
});

// Start the Server
app.listen(3000, () => {
  console.log("Server is Open!");
});
```

# Step-by-Step Walkthrough: The Event Loop in Express.js

This guide follows the "Waiter" (Main Thread) through three specific scenarios in an Express.js application to demonstrate how the Event Loop handles synchronous, asynchronous, and blocking code.

---

## 🛑 Phase 0: The Setup (Startup)

- **Action:** You run the command `node app.js`.
- **The Waiter (Main Thread):**
  1.  Reads the menu (Compiles the JavaScript code).
  2.  Sets up the kitchen stations (Registers the routes `/`, `/read-file`, `/block-me`).
  3.  Opens the front door (Starts listening on port 3000).
- **Current State:** The Waiter is now standing in the **Poll Phase**. His hands are empty. He is just waiting for a customer.

---

## 🚀 Scenario 1: The "Fast Food" Route (Synchronous)

**User visits:** `GET /`

1.  **Event:** A request arrives at the door.
2.  **Poll Phase:** The Waiter sees the request in the queue and picks it up immediately.
3.  **Execution:** He runs the callback function defined for `/`.
    - `console.log('A. Fast Route Hit')` runs.
    - `res.send('Hello World!')` runs.
4.  **Completion:** The function finishes. The Waiter returns to the **Poll Phase** to wait for the next person.
    - _Time taken:_ ~0.1ms (Instant).

---

## ⏳ Scenario 2: The "Slow Cooked" Route (Asynchronous I/O)

**User visits:** `GET /read-file`

### Part A: The Delegation

1.  **Event:** A request arrives.
2.  **Execution:** The Waiter picks it up.
    - He prints: `B. Reading File Started`.
    - He encounters `fs.readFile`.
3.  **The Handoff:**
    - The Waiter sees this is an **I/O task**. He does **not** do it himself.
    - He writes a "ticket" for the **System Kernel (Kitchen Staff)**: _"Please read big-file.txt and let me know when you are done."_
    - He registers a **callback function** (what to do when the file is ready) and puts it aside.
4.  **Moving On:**
    - He immediately moves to the next line of code.
    - He prints: `D. Reading File Delegated`.
5.  **Free Again:** His hands are empty! He goes back to the **Poll Phase**. He is now free to serve other users (like Scenario 1) while the file is being read in the background.

### Part B: The Return

1.  **The Kitchen:** The hard drive finishes reading the file. The Kernel puts the data into the **Poll Queue**.
2.  **The Waiter:**
    - Finishes whatever tiny task he was doing.
    - Checks the queue: "Oh, the file for Scenario 2 is ready!"
3.  **Callback Execution:**
    - He runs the specific callback he saved earlier.
    - He prints: `C. File Read Complete`.
    - He sends the file content to the user.

---

## ⛔ Scenario 3: The "Bad Chef" Route (CPU Blocking)

**User visits:** `GET /block-me`

1.  **Event:** A request arrives.
2.  **Execution:** The Waiter picks it up.
    - He prints: `E. Blocking Route Hit`.
3.  **The Trap (Blocking the Loop):**
    - He enters a `while` loop that counts from 0 to 5,000,000,000.
    - This is **CPU work**, not I/O work. He cannot delegate this. He must count every number himself.
4.  **The Crisis:**
    - While he is counting (which takes ~10 seconds), **he cannot move**.
    - **User A hits `/`:** The request sits in the queue. The Waiter ignores it.
    - **User B hits `/read-file`:** The request sits in the queue. The Waiter ignores it.
    - **The Kitchen:** Finishes a file read and rings the bell. The Waiter ignores it.
5.  **The Release:**
    - Finally, `count` reaches 5 billion.
    - He prints: `F. Blocking Finished`.
    - He sends the response: "Done!".
6.  **The Aftermath:**
    - Now that he is free, he rushes to the **Poll Queue** to handle all the angry customers (User A and User B) who were waiting while he was counting.

---

## 🔑 Key Takeaways

| Action                | Is the Waiter Blocked? | Can he serve others?                   |
| :-------------------- | :--------------------- | :------------------------------------- |
| **console.log**       | No (Instant)           | Yes                                    |
| **fs.readFile**       | No (Delegated)         | **YES** (This is the magic of Node.js) |
| **while loop / Math** | **YES**                | **NO** (The server freezes)            |

```

```
