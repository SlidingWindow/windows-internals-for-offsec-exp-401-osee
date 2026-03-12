# Critical Region in Windows / Operating System

## Definition

A **Critical Region (or Critical Section)** is a part of a program where shared resources are accessed.
Only **one thread at a time** is allowed to execute this part to prevent data corruption.

If multiple threads enter the critical region at the same time, it may cause:

* Data inconsistency
* Race condition
* Program crash
* Unexpected results

---

## Why Critical Region is needed

In multithreading, many threads may try to access the same variable or resource.

### Without Critical Region

```
Thread 1 -----------\
                     >---- Shared Data ----> ERROR / CORRUPTION
Thread 2 -----------/
```

Example:

```
Thread 1 writes value = 10
Thread 2 writes value = 20
Final value = unknown
```

This problem is called **Race Condition**.

---

## With Critical Region (Using Lock)

```
Thread 1 ---> [ LOCK ] ---> Critical Region ---> [ UNLOCK ] ---> Done
Thread 2 -------- waits --------> [ LOCK ] ---> Critical Region
```

Only one thread can enter at a time.

---

## Box Diagram

```
            +----------------------+
Thread 1 -->|                      |
            |   CRITICAL REGION    |--> Shared Data
Thread 2 -->|   (Protected Code)   |
            |                      |
            +----------^-----------+
                       |
                    LOCK / UNLOCK
```

---

## Windows Implementation (Critical Section)

In Windows, critical regions are implemented using **Critical Section objects**.

Functions used:

```
InitializeCriticalSection()
EnterCriticalSection()
LeaveCriticalSection()
DeleteCriticalSection()
```

Example (C / Win32)

```
CRITICAL_SECTION cs;

InitializeCriticalSection(&cs);

EnterCriticalSection(&cs);
// Critical Region Code
LeaveCriticalSection(&cs);

DeleteCriticalSection(&cs);
```

---

## Related Synchronization Methods

| Method           | Description                           |
| ---------------- | ------------------------------------- |
| Critical Section | Fast lock for threads in same process |
| Mutex            | Works between processes               |
| Semaphore        | Allows limited number of threads      |
| Spinlock         | Busy wait lock (kernel level)         |

---

## Real Life Example

Think of a bathroom with one key.

```
Person A → takes key → enters → leaves → returns key
Person B → waits until key is free
```

| Real Life | Program         |
| --------- | --------------- |
| Key       | Lock            |
| Bathroom  | Critical Region |
| People    | Threads         |

---

## Simple Definition (Exam Ready)

A **Critical Region** is a section of code that accesses shared data and must be executed by only one thread at a time to avoid race conditions.
