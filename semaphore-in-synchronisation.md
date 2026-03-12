# Semaphore – Operating System Synchronization

## Definition

A **Semaphore** is a synchronization object used to control access to a shared resource by multiple threads or processes.

Unlike a mutex, a semaphore can allow **more than one thread** to enter the critical region at the same time.

A semaphore keeps a **counter** that represents how many threads are allowed to access the resource.

---

## Simple Meaning

* Mutex → only 1 allowed
* Semaphore → limited number allowed

Example:

```text id="s1"
Semaphore count = 3
```

Up to 3 threads can enter at the same time.

---

## Why Semaphore is needed

When multiple threads share a resource that can handle more than one user.

Examples:

* Database connections
* Printer pool
* File readers
* Network connections
* Thread pool

---

## How Semaphore works

Semaphore has two main operations:

```text id="s2"
wait   (P operation)
signal (V operation)
```

Meaning:

| Operation | Meaning          |
| --------- | ---------------- |
| wait      | decrease counter |
| signal    | increase counter |

If counter = 0 → thread must wait

---

## Diagram – Semaphore

```id="s3"
          Semaphore = 2

Thread A ---> enter
Thread B ---> enter
Thread C ---> wait
Thread D ---> wait
```

Only 2 allowed at the same time.

---

## Critical Region with Semaphore

```id="s4"
wait(semaphore)

critical region

signal(semaphore)
```

This protects shared resources.

---

## Difference Between Mutex and Semaphore

| Feature         | Mutex         | Semaphore      |
| --------------- | ------------- | -------------- |
| Allows how many | 1             | many           |
| Has counter     | No            | Yes            |
| Ownership       | Yes           | No             |
| Use case        | single access | limited access |

---

## Types of Semaphore

### 1. Binary Semaphore

Only 0 or 1

Same as mutex, well **not** really! See the notes in the end for more info.

```text id="s5"
value = 0 or 1
```

---

### 2. Counting Semaphore

Allows multiple threads.

```text id="s6"
value = 0,1,2,3,4...
```

Used for resource pools.

---

## Example Scenario

Printer room with 3 printers.

```id="s7"
Semaphore = 3

User 1 → print
User 2 → print
User 3 → print
User 4 → wait
```

Only 3 can print at once.

---

## Windows Semaphore Functions

```text id="s8"
CreateSemaphore
WaitForSingleObject
ReleaseSemaphore
CloseHandle
```

Example flow:

```id="s9"
CreateSemaphore

WaitForSingleObject

use resource

ReleaseSemaphore
```

---

## Definition

A semaphore is a synchronization mechanism that uses a counter to control access to shared resources and allows multiple threads to enter the critical region up to a specified limit.

---

## One Line Summary

Semaphore allows a limited number of threads to access a shared resource at the same time.

---

## Ownership in Mutex vs Semaphore

When comparing mutex and semaphore, an important difference is **ownership**.

Ownership means that the thread that locks the object must also unlock it.

---

### Mutex Ownership

A mutex always has an owner.

```text
Thread A → lock mutex
Thread B → cannot unlock mutex
Thread A → must unlock mutex
```

Rules:

* Only one thread can lock it
* The same thread must release it
* Used for protecting critical regions safely

---

### Semaphore Has No Ownership

A semaphore does not track which thread entered the critical region.

```text
Thread A → wait(semaphore)
Thread B → release(semaphore)
```

This is allowed.

Rules:

* Uses a counter
* No owner
* Any thread can release
* Can allow multiple threads

---

### Binary Semaphore vs Mutex

A binary semaphore has values:

```text
0 or 1
```

Mutex also allows only one thread, but they are not the same.

| Feature                   | Mutex      | Binary Semaphore |
| ------------------------- | ---------- | ---------------- |
| Maximum value             | 1          | 1                |
| Ownership                 | Yes        | No               |
| Who can release           | Owner only | Any thread       |
| Safer for critical region | Yes        | Less safe        |

Because of ownership, mutex is usually preferred for critical region protection.

---

### Simple Analogy

Mutex = personal key lock

```text
You lock → only you unlock
```

Semaphore = token counter

```text
Take token
Return token
Anyone can return
```

---

### Important Note

Binary semaphore behaves like a mutex in simple cases, but internally they are different because a semaphore does not have ownership.
