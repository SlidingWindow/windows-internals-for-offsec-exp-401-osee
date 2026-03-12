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

## Simple Definition

A **Critical Region** is a section of code that accesses shared data and must be executed by only one thread at a time to avoid race conditions.

---

# Lab – Process Synchronization Using Mutex

## Goal

This lab demonstrates how to synchronize two processes that access the same file using a **mutex** to protect a **critical region**.

Two program instances will write to the same file.
Synchronization ensures the file is not corrupted.

---

## Shared Resource

File used in this lab:

```text
D:\logs\output.txt
```

Both processes will write into this file.

---

## Concepts Used

* Critical Region
* Mutex
* Process synchronization
* Wait / Lock
* Release / Unlock
* Shared file access

---

## Critical Region in this program

The critical region is the part of code that writes to the file.

```
lock mutex

write to file

unlock mutex
```

Only one process must execute this at a time.

---

## Diagram – With Mutex

```
Program A ----\
               >---[ LOCK ]--- Critical Region ---[ UNLOCK ]--- File
Program B ----/
```

Only one program writes at a time.

Example result:

```
PID 3201
PID 3201
PID 8744
PID 8744
PID 3201
```

Correct output.

---

## Diagram – Without Mutex

```
Program A ----\
               >---- Writing at same time ----> File damaged
Program B ----/
```

Example result:

```
PI32PID 874D 3201P44ID 32
```

Output is mixed → race condition.

---

## Steps

### 1. Create console project

```
Name: FileSyncTest
```

---

### 2. Create mutex

```
CreateMutex(
    NULL,
    FALSE,
    "FileWriteLock"
);
```

This mutex will be shared between processes.

---

### 3. Open or create file

```
D:\logs\output.txt
```

Create folder if needed.

---

### 4. Loop many times

Each loop writes the process id.

```
GetCurrentProcessId()
```

---

### 5. Protect critical region

```
WaitForSingleObject(lock, INFINITE)

write to file

ReleaseMutex(lock)
```

This ensures only one process writes at a time.

---

## Why waiting is needed

```
WaitForSingleObject → take lock
ReleaseMutex → free lock
```

If lock is busy → process waits.

---

## Run two programs

Run twice:

```
FileSyncTest.exe
FileSyncTest.exe
```

Both write to same file.

---

## Result with mutex

* No broken text
* Safe writing
* Correct order

---

## Result without mutex

Remove lock/unlock code and run again.

Result:

* mixed text
* corrupted lines
* race condition

This proves the need for a critical region.

---


