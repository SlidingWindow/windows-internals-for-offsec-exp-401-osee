In Windows kernel programming, a **Fast Mutex** is a synchronization object used to **protect shared data in kernel mode**, similar to a normal mutex, but designed to be **faster and lighter weight** for the Windows kernel.

It is used a lot inside Windows internals (EPROCESS, drivers, memory manager, etc.), so you’ll see things like:

```c
FAST_MUTEX Mutex;
ExAcquireFastMutex(&Mutex);
ExReleaseFastMutex(&Mutex);
```

Let’s break it down properly.

---

# 1️⃣ What problem Fast Mutex solves

When multiple threads run in kernel mode, they may access the same data:

```
Thread 1 → modifies structure
Thread 2 → reads structure
Thread 3 → deletes structure
```

Without synchronization → crash / corruption / BSOD.

So Windows provides locks:

| Lock type      | Where used                     |
| -------------- | ------------------------------ |
| Spinlock       | very short, high IRQL          |
| Fast Mutex     | medium duration, PASSIVE_LEVEL |
| Mutex / KMUTEX | heavier, slower                |
| ERESOURCE      | reader/writer lock             |
| Push Lock      | modern lightweight lock        |

Fast mutex is in the middle.

---

# 2️⃣ What is a Fast Mutex

A Fast Mutex is a kernel synchronization object defined as:

```c
typedef struct _FAST_MUTEX FAST_MUTEX, *PFAST_MUTEX;
```

Used with these APIs:

```
ExInitializeFastMutex
ExAcquireFastMutex
ExReleaseFastMutex
ExTryToAcquireFastMutex
```

Example:

```c
FAST_MUTEX MyMutex;

ExInitializeFastMutex(&MyMutex);

ExAcquireFastMutex(&MyMutex);

// protected code

ExReleaseFastMutex(&MyMutex);
```

---

# 3️⃣ Why it is called "Fast"

Compared to a normal kernel mutex (`KMUTEX`):

* Less overhead
* Simpler implementation
* No recursion
* No user-mode waiting
* Only kernel mode

So it is faster.

Windows uses it internally for:

* Memory manager
* Object manager
* I/O manager
* Driver code
* EPROCESS / ETHREAD fields

---

# 4️⃣ Important rules for Fast Mutex

This is VERY important in Windows internals.

### ✅ Must be at PASSIVE_LEVEL

You cannot acquire a fast mutex at high IRQL.

```
IRQL must be <= APC_LEVEL
```

Actually:

```
ExAcquireFastMutex → raises IRQL to APC_LEVEL
```

This prevents APCs while holding the lock.

So:

| Allowed IRQL   | Yes        |
| -------------- | ---------- |
| PASSIVE_LEVEL  | ✅          |
| APC_LEVEL      | ⚠️ careful |
| DISPATCH_LEVEL | ❌          |
| HIGH_LEVEL     | ❌          |

This is why drivers must be careful.

---

# 5️⃣ What happens internally

Fast mutex internally uses:

* Owner thread
* Count
* Event / wait block
* IRQL raise

Typical structure (simplified):

```c
typedef struct _FAST_MUTEX {
    LONG Count;
    PKTHREAD Owner;
    ULONG Contention;
    KEVENT Event;
} FAST_MUTEX;
```

(Not exact, but close)

When locked:

```
Count = 0 → locked
Count = 1 → free
```

If busy → thread waits.

---

# 6️⃣ Fast Mutex vs Spinlock vs Mutex

| Feature       | Spinlock       | Fast Mutex  | KMUTEX  |
| ------------- | -------------- | ----------- | ------- |
| IRQL          | DISPATCH_LEVEL | PASSIVE/APC | PASSIVE |
| Wait allowed  | ❌              | ✅           | ✅       |
| Fast          | ✅              | ✅           | ❌       |
| Kernel only   | ✅              | ✅           | mostly  |
| Recursion     | ❌              | ❌           | ✅       |
| Sleep allowed | ❌              | ✅           | ✅       |

Rule of thumb:

```
Very short → spinlock
Short/medium → fast mutex
Long → mutex / resource
```

---

# 7️⃣ Where you may have seen it

In Windows structures:

```
dt nt!_EPROCESS
dt nt!_MM_SESSION_SPACE
dt nt!_OBJECT_TYPE
```

You may see fields like:

```
_FAST_MUTEX
EX_PUSH_LOCK
ERESOURCE
```

These are all synchronization objects.

---

# 8️⃣ Why Windows kernel uses Fast Mutex a lot

Because kernel code often needs:

* safe
* blocking allowed
* but not too slow

Fast mutex gives that balance.

---

# 9️⃣ Important

```
ExAcquireFastMutex raises IRQL to APC_LEVEL
```

This is a key difference from KMUTEX.

Reason:

```
Prevent APC while holding lock
```

This matters in kernel debugging.

---

# 10️⃣ Summary

Fast Mutex =

> A lightweight kernel synchronization lock used at PASSIVE_LEVEL / APC_LEVEL to protect shared data, faster than KMUTEX but slower than spinlock.

---
