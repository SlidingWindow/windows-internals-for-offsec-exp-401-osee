
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





