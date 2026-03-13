## Why do we have so many synchronisation mechanisms in Windows?

Windows has **many synchronization mechanisms**, and each exists because the kernel needs locks that work at **different IRQL levels, different wait conditions, and different performance costs**.

Fast Mutex is just one of them, sitting somewhere in the middle.

Let’s compare them properly.

---

# Big Picture — Windows synchronization families

Windows kernel synchronization tools can be grouped like this:

| Category           | Examples                       | Purpose                             |
| ------------------ | ------------------------------ | ----------------------------------- |
| Atomic ops         | Interlocked*                   | very fast, no waiting               |
| Spin-based locks   | Spinlock, queued spinlock      | high IRQL, very short               |
| Lightweight locks  | Fast Mutex, Push Lock          | medium duration                     |
| Dispatcher objects | Mutex, Event, Semaphore, Timer | waitable objects                    |
| Executive locks    | ERESOURCE                      | reader/writer                       |
| APC mechanism      | APC                            | async execution, not exactly a lock |

Fast Mutex belongs to:

```
Lightweight kernel locks
```

---

# 1️⃣ Interlocked functions

Examples:

```
InterlockedIncrement
InterlockedDecrement
InterlockedCompareExchange
InterlockedExchange
```

### What they do

Atomic CPU instructions.

No waiting.
No blocking.
No scheduler.

Example:

```c
InterlockedIncrement(&Counter);
```

### Properties

| Feature              | Interlocked |
| -------------------- | ----------- |
| Fastest              | ✅           |
| Wait allowed         | ❌           |
| IRQL                 | any         |
| Sleep                | ❌           |
| Lock multiple fields | ❌           |

Use when:

```
Only one variable needs protection
```

Fast mutex is slower but can protect complex code.

---

# 2️⃣ Spinlocks

Used when:

```
High IRQL
Very short critical section
```

Example:

```
KeAcquireSpinLock
KeReleaseSpinLock
```

Properties:

| Feature            | Spinlock       |
| ------------------ | -------------- |
| IRQL               | DISPATCH_LEVEL |
| Sleep              | ❌              |
| Wait               | busy wait      |
| Fast               | very           |
| Good for long code | ❌              |

Fast mutex vs spinlock:

| Feature   | Fast Mutex  | Spinlock |
| --------- | ----------- | -------- |
| IRQL      | PASSIVE/APC | DISPATCH |
| Sleep     | ✅           | ❌        |
| Busy wait | ❌           | ✅        |
| Long code | OK          | bad      |

---

# 3️⃣ Fast Mutex

```
ExAcquireFastMutex
ExReleaseFastMutex
```

Properties:

| Feature            | Fast Mutex   |
| ------------------ | ------------ |
| IRQL               | <= APC_LEVEL |
| Sleep allowed      | ✅            |
| Waitable           | internally   |
| Recursive          | ❌            |
| Kernel only        | ✅            |
| Faster than KMUTEX | ✅            |

Used for:

```
Medium length kernel critical sections
```

---

# 4️⃣ Kernel Dispatcher Objects

These are real kernel objects.

Examples:

* Mutex (KMUTEX)
* Semaphore
* Event
* Timer
* Thread
* Process

They use:

```
KeWaitForSingleObject
KeWaitForMultipleObjects
```

Properties:

| Feature            | Dispatcher object |
| ------------------ | ----------------- |
| Waitable           | ✅                 |
| Scheduler involved | ✅                 |
| Heavy              | yes               |
| User mode usable   | yes               |
| Kernel usable      | yes               |

Example:

```c
KeWaitForSingleObject(&Mutex, ...);
```

Fast mutex is lighter because:

```
No full dispatcher wait
```

---

# 5️⃣ Mutex (KMUTEX)

Kernel mutex object.

Slower but more flexible.

Properties:

| Feature              | KMUTEX  |
| -------------------- | ------- |
| Waitable             | ✅       |
| Recursive            | ✅       |
| User-mode compatible | partly  |
| Slow                 | yes     |
| IRQL                 | PASSIVE |

Fast mutex vs KMUTEX:

| Feature    | Fast   | KMUTEX |
| ---------- | ------ | ------ |
| Speed      | faster | slower |
| Recursion  | ❌      | ✅      |
| Dispatcher | no     | yes    |
| Simpler    | yes    | no     |

---

# 6️⃣ Semaphore

Used for:

```
Allow N threads
```

Example:

```
KeReleaseSemaphore
KeWaitForSingleObject
```

Not a mutex.

Used for:

* queues
* resource limits
* worker threads

Fast mutex allows only:

```
1 owner
```

---

# 7️⃣ Events

Used for signaling.

```
KeSetEvent
KeWaitForSingleObject
```

Example:

```
Thread waits until something happens
```

Not a lock.

Used for:

* I/O completion
* driver sync
* thread wakeup

Fast mutex is for:

```
Mutual exclusion
```

---

# 8️⃣ Executive Resource (ERESOURCE)

Reader / writer lock.

```
ExAcquireResourceShared
ExAcquireResourceExclusive
```

Properties:

| Feature          | ERESOURCE |
| ---------------- | --------- |
| Multiple readers | ✅         |
| Single writer    | ✅         |
| Slow             | yes       |
| Complex          | yes       |

Used in:

* file system
* memory manager
* object manager

Fast mutex is simpler:

```
Only one owner
```

---

# 9️⃣ APC (Asynchronous Procedure Call)

Not really a lock.

APC = run code later in thread context.

Used for:

* I/O completion
* thread callbacks
* kernel async work

Fast mutex actually blocks APC:

```
ExAcquireFastMutex → raises IRQL to APC_LEVEL
```

So APC cannot run.

This is important.

---

# 🔟 Best comparison table

| Mechanism   | Speed     | Sleep | IRQL     | Waitable | Use case        |
| ----------- | --------- | ----- | -------- | -------- | --------------- |
| Interlocked | fastest   | ❌     | any      | ❌        | atomic ops      |
| Spinlock    | very fast | ❌     | DISPATCH | ❌        | short high IRQL |
| Fast Mutex  | fast      | ✅     | <= APC   | no       | medium kernel   |
| Push Lock   | fast      | ✅     | <= APC   | no       | modern kernel   |
| KMUTEX      | slow      | ✅     | PASSIVE  | yes      | heavy sync      |
| Semaphore   | medium    | ✅     | PASSIVE  | yes      | counting        |
| Event       | medium    | ✅     | PASSIVE  | yes      | signaling       |
| ERESOURCE   | slow      | ✅     | PASSIVE  | no       | RW lock         |
| APC         | n/a       | n/a   | thread   | n/a      | async call      |

---

# Why Fast Mutex exists

Because Windows needed something:

```
Not as dangerous as spinlock
Not as heavy as mutex
Safe to block
Kernel only
```

So they made:

```
FAST_MUTEX
```

---

