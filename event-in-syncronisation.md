# Event – Windows Synchronization Object

## Definition

An **Event** is a synchronization object in Windows that allows **threads to wait for a signal** before continuing execution.

It’s commonly used for **thread coordination**, signaling that some work is complete, or controlling the sequence of execution.

---

## Types of Events

### 1. Manual-Reset Event

* Stays signaled until explicitly **reset**.
* All waiting threads are released when signaled.
* Example:

```text id="ev1"
Event state: SIGNALLED (all waiting threads proceed)
ResetEvent() → back to NON-SIGNALLED
```

### 2. Auto-Reset Event

* Automatically returns to non-signaled after **releasing one waiting thread**.
* Only one thread proceeds each time it’s signaled.
* Example:

```text id="ev2"
Event state: SIGNALLED
WaitOne thread released → event resets automatically
```

---

## How Events Work

* Thread waits on an event using:

```text id="ev3"
WaitForSingleObject(eventHandle, INFINITE)
```

* Another thread signals the event using:

```text id="ev4"
SetEvent(eventHandle)
```

* For manual-reset events, multiple threads waiting simultaneously will all be released.
* For auto-reset events, only **one thread** is released at a time.

---

## Real-World Analogy

* **Manual-Reset Event:** A traffic light turns green → all cars waiting go at once.
* **Auto-Reset Event:** A single-lane bridge light → only one car passes each green signal.

---

## Windows Functions for Events

| Function            | Purpose                                   |
| ------------------- | ----------------------------------------- |
| CreateEvent         | Creates manual or auto-reset event        |
| SetEvent            | Signals event (makes it signaled)         |
| ResetEvent          | Resets manual-reset event to non-signaled |
| WaitForSingleObject | Thread waits for event                    |
| CloseHandle         | Clean up event handle                     |

---

## Critical Region vs Event

* Events don’t directly protect shared resources.
* They are used for **thread coordination**, e.g., "do not start until initialization is complete."
* Can be used in combination with **mutex/semaphore**.

---

## Example Scenario

Imagine a thread pool:

1. Worker threads wait for a signal (event) to start work.
2. Main thread sets the event when tasks are ready.
3. Workers proceed simultaneously (manual-reset) or one at a time (auto-reset).

---

## Summary

* **Event** = a signaling object that threads wait on.
* **Manual-reset** → signal stays; multiple threads can proceed.
* **Auto-reset** → resets automatically; only one thread proceeds per signal.
* Used for **thread coordination**, not mutual exclusion.

---

### Note

> Use **mutex** for exclusive access, **semaphore** for limited concurrent access, and **events** to **signal or coordinate threads**.

---

Simple C program to demonstrate real world usage of Event.

```C
// File: EventDemo.cpp
#include <windows.h>
#include <iostream>
#include <thread>
#include <vector>

using namespace std;

// Event handles
HANDLE g_manualEvent;
HANDLE g_autoEvent;

// Worker function for manual-reset event
void workerManual(int id) {
    cout << "Thread " << id << " waiting for manual-reset event...\n";
    WaitForSingleObject(g_manualEvent, INFINITE);
    cout << "Thread " << id << " proceeding after manual-reset event.\n";
}

// Worker function for auto-reset event
void workerAuto(int id) {
    cout << "Thread " << id << " waiting for auto-reset event...\n";
    WaitForSingleObject(g_autoEvent, INFINITE);
    cout << "Thread " << id << " proceeding after auto-reset event.\n";
}

int main() {
    cout << "=== Manual-Reset Event Demo ===\n";

    // Create manual-reset event (initially non-signaled)
    g_manualEvent = CreateEvent(NULL, TRUE, FALSE, NULL);

    vector<thread> threads;
    for (int i = 1; i <= 3; i++)
        threads.push_back(thread(workerManual, i));

    Sleep(1000); // simulate setup time
    cout << "Main thread signals manual-reset event!\n";
    SetEvent(g_manualEvent); // all waiting threads proceed

    for (auto& t : threads) t.join();
    threads.clear();

    // Reset for auto-reset demo
    cout << "\n=== Auto-Reset Event Demo ===\n";
    g_autoEvent = CreateEvent(NULL, FALSE, FALSE, NULL); // auto-reset

    for (int i = 1; i <= 3; i++)
        threads.push_back(thread(workerAuto, i));

    Sleep(1000);
    for (int i = 1; i <= 3; i++) {
        cout << "Main thread signals auto-reset event!\n";
        SetEvent(g_autoEvent); // releases one thread at a time
        Sleep(500); // small delay to show sequential release
    }

    for (auto& t : threads) t.join();

    // Cleanup
    CloseHandle(g_manualEvent);
    CloseHandle(g_autoEvent);

    return 0;
}
```
