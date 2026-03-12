
# Asynchronous Procedure Calls (APCs) – Windows

---

## Definition

An **Asynchronous Procedure Call (APC)** is a function that executes **asynchronously in the context of a specific thread**.  

- APCs allow a thread to **execute a function when it enters an alertable state**.  
- Useful for **asynchronous I/O, timers, or deferred work**.  

---

## Key Concepts

1. **Thread-specific**: APCs are queued to a particular thread.  
2. **Alertable state**: A thread must enter an alertable wait to execute queued APCs.  
   - Functions like `SleepEx`, `WaitForSingleObjectEx`, `WaitForMultipleObjectsEx` with `bAlertable = TRUE`.  
3. **Asynchronous**: The function executes **outside of the current execution path**, but **in the target thread**.  

---

## Types of APCs

### 1. User-mode APC
- Queued with `QueueUserAPC`  
- Executed in **user-mode**  
- Requires the target thread to be in **alertable wait**  

### 2. Kernel-mode APC
- Queued by kernel (e.g., I/O completion)  
- Executed in **kernel-mode**  

---

## Windows Functions

| Function | Purpose |
|----------|---------|
| `QueueUserAPC` | Queue a user-mode APC to a thread |
| `SleepEx` | Waits alertably for a time period and executes queued APCs |
| `WaitForSingleObjectEx` | Alertable wait for handle signal and APC execution |
| `WaitForMultipleObjectsEx` | Alertable wait for multiple handles and APC execution |

---

## Usage Example (C++ / Win32)

```cpp
#include <windows.h>
#include <iostream>

using namespace std;

// APC callback function
VOID CALLBACK MyAPC(ULONG_PTR dwParam) {
    cout << "APC executed with param: " << dwParam << endl;
}

DWORD WINAPI ThreadFunc(LPVOID lpParam) {
    cout << "Thread entering alertable wait..." << endl;
    SleepEx(5000, TRUE); // alertable wait for 5 seconds
    cout << "Thread finished waiting." << endl;
    return 0;
}

int main() {
    HANDLE hThread = CreateThread(NULL, 0, ThreadFunc, NULL, 0, NULL);

    // Queue APC to the thread
    QueueUserAPC(MyAPC, hThread, 42);

    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);

    return 0;
}
````

---

## How It Works

1. Main thread creates a worker thread.
2. `QueueUserAPC` queues `MyAPC` to the worker thread.
3. Worker thread calls `SleepEx` with `alertable = TRUE`.
4. When the thread enters alertable wait, the queued APC executes.

---

## Real-World Analogy

* APC = **“reminder note” for a thread**
* Thread sees the note only when it **pauses to check messages** (alertable wait)
* Allows **asynchronous notifications without polling**

---

## Key Points

* APCs **execute in the context of a specific thread**.
* **Thread must be alertable**; otherwise, APCs wait indefinitely.
* Useful for **asynchronous I/O, timers, or deferred work**.
* Kernel-mode APCs are automatically executed by the system for I/O completion.

---

### Note

> Think APC = “queue a task for a thread, executed when the thread is ready (alertable)”.
> Useful for asynchronous I/O or callback scenarios in Windows.

---

Ah! Now we’re getting into the **practical side of APCs**. Windows uses APCs internally in several situations, mainly to let threads handle **asynchronous work or deferred callbacks** safely. Here’s a clear breakdown:

---

## When Windows Uses APCs

### 1. **Asynchronous I/O**

* When you perform asynchronous I/O (`ReadFileEx`, `WriteFileEx`) in Windows, the **kernel queues a user-mode APC** to the thread that initiated the I/O.
* The APC executes **when the thread enters an alertable wait**, letting the thread process I/O completion **without polling**.

**Example:**

```cpp
ReadFileEx(hFile, buffer, size, &overlapped, MyAPC); 
```

* `MyAPC` will be called once the read is complete, **while the thread is waiting alertably**.

---

### 2. **Thread Notifications / Deferred Work**

* Windows kernel can queue APCs to notify threads of **system events** or **completion of some kernel work**.
* APC allows these notifications to happen **in the thread context** rather than in arbitrary kernel threads, which is safer.

---

### 3. **Timers**

* APCs are used by **timer objects** that execute callbacks after a specified timeout.
* Functions like `CreateTimerQueueTimer` can queue an APC when the timer expires.

---

### 4. **Kernel-Mode APCs**

* Certain **kernel operations** automatically queue **kernel-mode APCs** to threads, e.g.:

  * Memory management events
  * File system I/O completions
  * APCs that run in kernel-mode (not user-mode) to notify the thread that a kernel resource is ready

> Note: Kernel-mode APCs are invisible to normal user-mode code. User-mode APCs require the thread to be in **alertable wait**.

---

### 5. **Thread Alertable Waits**

* APCs only execute when a thread calls a function like:

  * `SleepEx(timeout, TRUE)`
  * `WaitForSingleObjectEx(handle, timeout, TRUE)`
  * `WaitForMultipleObjectsEx(handles, timeout, TRUE)`

This is why APCs are **deferred** — they wait until the thread voluntarily enters an **alertable state**.

---

### Real-World Analogy

* APC = “post-it note for a thread”:

  * Thread does other work normally.
  * When it **pauses to check messages** (alertable wait), the note (APC) executes.
* It’s like scheduling a small task **without interrupting the thread**, but guaranteeing it will execute **in the thread’s own context**.

---

### Quick Summary Table

| APC Type        | Queued By                  | Executes When                    | Use Case                                                |
| --------------- | -------------------------- | -------------------------------- | ------------------------------------------------------- |
| User-mode APC   | `QueueUserAPC` / async I/O | Thread enters alertable wait     | Asynchronous I/O callbacks, deferred work               |
| Kernel-mode APC | Kernel                     | Thread executes kernel APC queue | I/O completions, memory notifications, driver callbacks |

---


---

Asynchronous Procedure Call (APC) – Visual Diagram

```
Time →
Main Thread:
┌────────────────────────────┐
│ Create Worker Thread        │
└────────────────────────────┘
│
▼
Worker Thread:
┌────────────────────────────┐
│ Running normally            │
└────────────────────────────┘
│
▼
┌────────────────────────────┐
│ Enter alertable wait        │  ← SleepEx(..., TRUE)
└────────────────────────────┘
│
▼
┌────────────────────────────┐
│ APC Queued (MyAPC)         │
│ Executed here automatically│
└────────────────────────────┘
│
▼
┌────────────────────────────┐
│ Continue normal execution  │
└────────────────────────────┘
```

### Notes:

- The **APC is queued** by `QueueUserAPC` from another thread.  
- The **worker thread executes the APC only when it enters an alertable wait** (`SleepEx`, `WaitForSingleObjectEx`, etc.).  
- This allows **asynchronous callbacks without polling**.  
- Multiple APCs can be queued; they execute in **FIFO order** when the thread is alertable.

---





