# Windows Synchronization Primitives – Study Notes

---

## 1. Mutex

### Definition

A **Mutex** (mutual exclusion) is a synchronization object that allows **only one thread** to access a critical region at a time.  

It has **ownership**, meaning only the thread that locks it can unlock it.

### Usage Example (C++ / Win32)

```cpp
HANDLE g_mutex = CreateMutex(NULL, FALSE, NULL);

WaitForSingleObject(g_mutex, INFINITE);
// Critical region: shared resource access
ReleaseMutex(g_mutex);

CloseHandle(g_mutex);
````

### Real-World Analogy

Imagine a bathroom with a personal key: only the person inside can unlock it.

---

## 2. Semaphore

### Definition

A **Semaphore** is a counter-based synchronization object that allows **N threads** to access a resource simultaneously.

It does **not have ownership**.

### Usage Example (C++ / Win32)

```cpp
HANDLE g_semaphore = CreateSemaphore(NULL, 2, 2, NULL); // max 2 threads

WaitForSingleObject(g_semaphore, INFINITE);
// Critical region: limited resource
ReleaseSemaphore(g_semaphore, 1, NULL);

CloseHandle(g_semaphore);
```

### Mutex vs Semaphore

```
| Feature     | Mutex            | Semaphore     |
| ----------- | ---------------- | ------------- |
| Max threads | 1                | N             |
| Ownership   | Yes              | No            |
| Use case    | Critical section | Resource pool |
```

### Real-World Analogy

Think of a parking lot with 5 spaces. Any car can leave to free a spot.

---

## 3. Event

### Definition

An **Event** is a signaling object that threads wait on.

It is used for **thread coordination**, not mutual exclusion.

### Types

* **Manual-reset event** → stays signaled until reset; all waiting threads proceed.
* **Auto-reset event** → resets automatically; only one thread proceeds per signal.

### Usage Example (C++ / Win32)

```cpp
HANDLE g_manualEvent = CreateEvent(NULL, TRUE, FALSE, NULL); // manual-reset
HANDLE g_autoEvent = CreateEvent(NULL, FALSE, FALSE, NULL);  // auto-reset

WaitForSingleObject(g_manualEvent, INFINITE);
SetEvent(g_manualEvent);
```

### Real-World Analogy

* Manual-reset event: like a traffic light turning green, all cars proceed.
* Auto-reset event: like a single-lane bridge, only one car passes at a time.

---

## 4. Executive Resource (ERESOURCE)

### Definition

An **Executive Resource** is a kernel-mode synchronization object that allows **shared (read) or exclusive (write) access**.

### Features

* Shared: multiple threads can read simultaneously.
* Exclusive: only one thread can write.
* Supports upgrade/downgrade between shared and exclusive access.

### Comparison

```
| Feature   | Mutex            | Semaphore         | Executive Resource                           |
| --------- | ---------------- | ----------------- | -------------------------------------------- |
| Access    | Exclusive        | Limited count     | Shared (read) or Exclusive (write)           |
| Ownership | Yes              | No                | Yes                                          |
| Wait      | Single thread    | Limited N threads | Shared threads coexist; exclusive blocks all |
| Use case  | Critical section | Resource pool     | Kernel read/write structures                 |
```

### ASCII Diagram – Shared Access

```
Time →
Thread 1 ──[READ]─────────────
Thread 2 ──[READ]─────────────
Thread 3 ──[READ]─────────────
Thread 4 ──[READ]─────────────
```

All threads can read simultaneously.

### ASCII Diagram – Exclusive Access

```
Time →
Thread 1 ──[WRITE]────────────
Thread 2 ── waits
Thread 3 ── waits
```

Only one thread can write at a time; others wait.

### ASCII Diagram – Mixed Access

```
Time →
Thread 1 ──[READ]─────────────
Thread 2 ──[READ]─────────────
Thread 3 ──[WRITE]── waits
Thread 4 ──[READ]── waits
Thread 3 ──[WRITE]─────────────
Thread 4 ──[READ]─────────────
```

Shared threads coexist; exclusive thread blocks others.

### Real-World Analogy

* Shared = library reading room (many readers).
* Exclusive = editing a manuscript (one writer).

---

## 5. Summary Table

```
| Object             | Access           | Ownership | Threads Allowed              | Use Case                |
| ------------------ | ---------------- | --------- | ---------------------------- | ----------------------- |
| Mutex              | Exclusive        | Yes       | 1                            | Critical section        |
| Binary Semaphore   | Exclusive        | No        | 1                            | Simple resource lock    |
| Counting Semaphore | Exclusive        | No        | N                            | Resource pool           |
| Event              | Signaling        | N/A       | All waiting                  | Thread coordination     |
| Executive Resource | Shared/Exclusive | Yes       | Multiple (read) or 1 (write) | Kernel-level read/write |
```
---

## 6. Code Examples Reference

* **Mutex & Semaphore Demo** → `SyncDemo.cpp`
* **Event Demo** → `EventDemo.cpp`
* **Executive Resource** → kernel-mode only


---

