---
layout: page
title: "Lab2-w5: uthread & Hash Table"
short_title: "W5: uthread & hash table"
lab: lab2
week: 5
order: 1
date: 2026-02-05
---

<div class="info-box">
📦 <strong>Lab zip:</strong> xv6labs-w5 &nbsp;|&nbsp; 📊 <strong>Score:</strong> 60/60 &nbsp;|&nbsp; 📅 <strong>Due:</strong> 20 Feb 2026 11:59pm
</div>

## Task 1 — `uthread` (User-Level Thread Context Switch)

### What you need to do

Complete the user-level threading library in `user/uthread.c` and `user/uthread_switch.S`:
1. Define `struct thread_context` to store RISC-V callee-saved registers
2. Embed it in `struct thread`
3. Implement `thread_create()` to set up the initial stack pointer and return address
4. Call `thread_switch` from `thread_schedule()`
5. Implement `thread_switch` in assembly to save/restore registers

### Registers to save/restore

RISC-V calling convention: only **callee-saved** registers must be preserved across a function call. These are: `ra`, `sp`, `s0`–`s11` (14 registers total).

| Register | Role |
|----------|------|
| `ra` | Return address — where to resume execution |
| `sp` | Stack pointer — each thread has its own stack |
| `s0`–`s11` | Saved registers — callee must restore these |

### Step 1: `struct thread_context` in `user/uthread.c`

```c
struct thread_context {
    uint64 ra;
    uint64 sp;
    /* callee-saved */
    uint64 s0;
    uint64 s1;
    uint64 s2;
    uint64 s3;
    uint64 s4;
    uint64 s5;
    uint64 s6;
    uint64 s7;
    uint64 s8;
    uint64 s9;
    uint64 s10;
    uint64 s11;
};

struct thread {
    char                stack[STACK_SIZE];
    int                 state;
    struct thread_context ctx;   // ADD THIS
};
```

### Step 2: `thread_create()`

The stack grows **downward**. `stack` is an array of `STACK_SIZE` bytes. The initial stack pointer must point to the **end** of the array (highest address).

```c
void
thread_create(void (*func)())
{
    struct thread *t;

    for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
        if (t->state == FREE) break;
    }
    t->state = RUNNABLE;

    // Stack grows down: SP starts at the TOP of the stack array
    t->ctx.sp = (uint64)(t->stack + STACK_SIZE);

    // RA = the function to run when this thread first executes
    t->ctx.ra = (uint64)func;
}
```

### Step 3: `thread_schedule()` — call `thread_switch`

Replace the `/* YOUR CODE HERE */` comment:

```c
thread_switch((uint64)&t->ctx, (uint64)&next_thread->ctx);
```

`thread_switch` takes two `uint64` arguments: pointer to the **old** context, pointer to the **new** context.

### Step 4: `user/uthread_switch.S`

Mirror the kernel's `kernel/swtch.S` but using `a0` (old ctx) and `a1` (new ctx):

```asm
    .text
    .globl thread_switch
thread_switch:
    /* Save callee-saved registers of the OLD thread into *a0 */
    sd ra,   0(a0)
    sd sp,   8(a0)
    sd s0,  16(a0)
    sd s1,  24(a0)
    sd s2,  32(a0)
    sd s3,  40(a0)
    sd s4,  48(a0)
    sd s5,  56(a0)
    sd s6,  64(a0)
    sd s7,  72(a0)
    sd s8,  80(a0)
    sd s9,  88(a0)
    sd s10, 96(a0)
    sd s11,104(a0)

    /* Restore callee-saved registers of the NEW thread from *a1 */
    ld ra,   0(a1)
    ld sp,   8(a1)
    ld s0,  16(a1)
    ld s1,  24(a1)
    ld s2,  32(a1)
    ld s3,  40(a1)
    ld s4,  48(a1)
    ld s5,  56(a1)
    ld s6,  64(a1)
    ld s7,  72(a1)
    ld s8,  80(a1)
    ld s9,  88(a1)
    ld s10, 96(a1)
    ld s11,104(a1)

    ret   /* jump to ra of the new thread */
```

### How `ret` works here

`ret` is just `jalr x0, ra, 0` — it jumps to whatever address is in `ra`. Since we just loaded `ra` from the new thread's context (which was set to `func` in `thread_create`), the CPU jumps into `thread_a`, `thread_b`, or `thread_c` on first run — or back to wherever the thread last called `thread_yield()` on subsequent runs.

---

## Task 2 — Multi-Threaded Hash Table (`ph`)

### Background

- `ph-without-locks.c`: correct single-threaded, crashes multi-threaded (data loss)
- `ph-with-mutex-locks.c`: your task — add **per-bucket** mutex locks

### Why data is lost without locks

When two threads call `put()` simultaneously on the same bucket, they both read the bucket head, both create a new node pointing to the old head, and both write back — the second write overwrites the first, losing one entry.

### Per-bucket locking strategy

Assign one `pthread_mutex_t` per bucket. Threads only block each other if they hash to the **same** bucket, allowing parallel insertions into different buckets.

### Solution (`ph-with-mutex-locks.c`)

```c
// Add at file scope (after NBUCKET define):
pthread_mutex_t bucket_locks[NBUCKET];

// In main(), before creating threads:
for (int i = 0; i < NBUCKET; i++)
    pthread_mutex_init(&bucket_locks[i], NULL);

// After joining all threads:
for (int i = 0; i < NBUCKET; i++)
    pthread_mutex_destroy(&bucket_locks[i]);
```

Modify `put()`:

```c
static void
put(int key, int value)
{
    int bucket = key % NBUCKET;

    pthread_mutex_lock(&bucket_locks[bucket]);

    // check if key already exists → update
    struct entry *e = table[bucket];
    for (; e != 0; e = e->next) {
        if (e->key == key) {
            e->value = value;
            pthread_mutex_unlock(&bucket_locks[bucket]);
            return;
        }
    }
    // insert at head
    insert(key, value, &table[bucket], table[bucket]);

    pthread_mutex_unlock(&bucket_locks[bucket]);
}
```

`get()` does **not** need a lock because:
1. All `put()`s complete before any `get()` starts (barrier in the harness)
2. `get()` only reads; no writes happen concurrently during the get phase

### Expected results

| Config | Missing keys | Throughput |
|--------|-------------|------------|
| No lock, 1 thread | 0 | ~20k/s |
| No lock, 2 threads | 600–1500 | ~42k/s |
| With lock, 1 thread | 0 | ~22k/s |
| With lock, 2 threads | **0** | ~33k/s ✓ |

---

## Variation Questions

<div class="var-question">
<div class="var-qnum">Q1 <span class="diff diff-easy">Easy</span></div>

In `thread_switch`, why do we save/restore `s0`–`s11` but **not** `a0`–`a7` or `t0`–`t6`?

<div class="answer-content">

<p>RISC-V calling convention divides registers into two categories:</p>

<ul>
  <li><strong>Caller-saved</strong> (<code>a0</code>–<code>a7</code>, <code>t0</code>–<code>t6</code>): The function that calls <code>thread_switch</code> is responsible for saving these before the call if it needs them afterward. From the compiler's perspective, any function call may clobber these registers.</li>
</ul>
<ul>
  <li><strong>Callee-saved</strong> (<code>s0</code>–<code>s11</code>, <code>ra</code>, <code>sp</code>): The called function (<code>thread_switch</code>) must restore these to their original values before returning. The compiler guarantees that after <code>thread_switch</code> returns, these have their original values.</li>
</ul>
Since threads yield at <code>thread_yield()</code> → <code>thread_schedule()</code> → <code>thread_switch()</code>, the compiler has already saved any caller-saved registers it cares about across that call chain. We only need to save what the compiler assumes <code>thread_switch</code> will preserve — the callee-saved set.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q2 <span class="diff diff-med">Medium</span></div>

What would happen if `thread_create` set `t->ctx.sp` to the **beginning** of the stack array (lowest address) instead of the end?

<div class="answer-content">

<p>The stack would grow <strong>downward into memory that precedes the stack array</strong> — i.e., into either the previous thread's stack (if the threads are adjacent in <code>all_thread[]</code>) or into the code/data segment. This would corrupt memory and cause undefined behavior.</p>

<p>Specifically: every function call pushes a stack frame by subtracting from <code>sp</code>. Starting at <code>stack[0]</code> means the first push writes to <code>stack[-8]</code>, which is outside the array — a buffer underflow bug.</p>

<p>Setting <code>sp = stack + STACK_SIZE</code> (pointing one past the end, but since the stack grows down, the first push goes to <code>stack + STACK_SIZE - 8</code> which is valid) is the correct initialization.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q3 <span class="diff diff-med">Medium</span></div>

In the hash table, why is a single global mutex (one lock for the whole table) **correct** but **slower** than per-bucket locking? Give a scenario where per-bucket locking provides no speedup over a single global lock.

<div class="answer-content">

<strong>Correct but slow:</strong> A single global mutex serializes all <code>put()</code> operations, regardless of which bucket they target. With 2 threads, even if they would hash to different buckets (and thus never conflict), they still wait for each other. This eliminates parallelism.

<strong>Per-bucket speedup:</strong> Thread A on bucket 0 and Thread B on bucket 3 can run simultaneously because they hold different locks.

<strong>When per-bucket gives no speedup:</strong> If all keys hash to the <strong>same bucket</strong> (e.g., all keys are multiples of <code>NBUCKET</code>), then all threads always contend for <code>bucket_locks[0]</code>. They serialize just as they would with a single global lock. This is called a "hot spot" or "lock contention" problem — the locks are too coarse when the workload is skewed.

<p>The fix: use a better hash function or increase <code>NBUCKET</code>.</p>

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q4 <span class="diff diff-hard">Hard</span></div>

Add a fifth thread type `thread_d` to the uthread system that runs a "daemon" — it never exits but instead calls `thread_yield()` in an infinite loop. What happens when threads a, b, c finish and only the daemon is left? How would you modify the system to handle this gracefully?

<div class="answer-content">

<p>With the current scheduler, when a, b, c all set <code>state = FREE</code> and call <code>thread_schedule()</code>, the scheduler finds only the daemon (<code>RUNNABLE</code>) and switches to it. The daemon runs indefinitely — <code>thread_schedule: no runnable threads</code> is never printed.</p>

<strong>Graceful termination options:</strong>

<ol>
  <li><strong>Count active threads:</strong> Add a global <code>int active_threads</code>. Decrement it before a thread calls <code>thread_schedule()</code> with <code>state = FREE</code>. Daemon checks <code>if (active_threads == 0) { current_thread->state = FREE; thread_schedule(); }</code>.</li>
</ol>
<ol>
  <li><strong>Sentinel thread:</strong> The main thread (thread 0) waits for all others to finish. Use a join-like mechanism: <code>thread_join(t)</code> that spins yielding until <code>t->state == FREE</code>.</li>
</ol>
<ol>
  <li><strong>Exit flag:</strong> Set a global <code>int all_done = 0</code>. When all worker threads finish, set <code>all_done = 1</code>. The daemon checks this flag each iteration and exits when set.</li>
</ol>
Option 1 is the most UNIX-like: it mirrors how the kernel tracks process count for <code>wait()</code>.

</div>
</div>

<div class="var-question">
<div class="var-qnum">Q5 <span class="diff diff-hard">Hard</span></div>

The `ph` hash table uses a **linked list** per bucket for collision resolution. Describe how you would convert it to use **open addressing** (linear probing) instead. What changes would be needed to ensure thread safety with the per-bucket lock approach?

<div class="answer-content">

<strong>Open addressing replaces linked lists with a flat array:</strong>

<pre><code class="language-c">struct entry table[NBUCKET * LOAD_FACTOR];  // flat array, not array of lists
</code></pre>

<p>On collision, probe the next slot: <code>(hash + i) % TOTAL_SIZE</code> until you find an empty slot.</p>

<strong>Thread safety complications:</strong>

<ol>
  <li><strong>Lock scope changes:</strong> With linked lists, a lock per bucket is clean. With linear probing, an insert might probe across bucket boundaries, requiring you to lock all potentially-touched buckets — or use a single global lock.</li>
</ol>
<ol>
  <li><strong>Deletion is hard:</strong> Removing an entry in linear probing requires either tombstone markers or rehashing all subsequent entries. Tombstones need to be checked during <code>get()</code>, complicating the read path.</li>
</ol>
<ol>
  <li><strong>Resize/rehash:</strong> When load factor exceeds ~70%, you rehash the entire table — which requires acquiring all locks simultaneously (or using a read-write lock and taking the write lock to rehash).</li>
</ol>
In practice, chaining (linked lists) is easier to make concurrent than open addressing because the lock scope is naturally limited to one bucket.

</div>
</div>
