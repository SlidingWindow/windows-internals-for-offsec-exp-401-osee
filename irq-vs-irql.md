# IRQs and IRQLs -> Are they different or same, or related to each other?

Yes — **IRQ** and **IRQL** are related but **not the same thing**. They come from different layers of how an OS handles interrupts.

---

## 1. IRQ (Interrupt Request)

**IRQ** is a **hardware signal** sent by a device to the CPU to request attention.

Examples:

* Keyboard pressed a key
* Network card received data
* Disk finished reading a sector

In traditional PC architecture, each device line is assigned an **IRQ number** (like IRQ0, IRQ1, etc.).

Example mapping (classic PCs):

* **IRQ0** – System timer
* **IRQ1** – Keyboard
* **IRQ14** – Primary disk controller

So in simple terms:

➡ **IRQ = a physical hardware interrupt line / signal**

---

## 2. IRQL (Interrupt Request Level)

**IRQL** is a **software priority level used by the Windows kernel** to control interrupt handling.

The Windows kernel (NT architecture) uses IRQLs to decide **what code can run and what interrupts can be processed** at any moment.

Some important Windows IRQL levels:

| IRQL           | Meaning                      |
| -------------- | ---------------------------- |
| PASSIVE_LEVEL  | Normal thread execution      |
| APC_LEVEL      | Asynchronous procedure calls |
| DISPATCH_LEVEL | Scheduler / spinlocks        |
| Device IRQLs   | Hardware interrupt handling  |
| HIGH_LEVEL     | Highest priority             |

When the system raises IRQL:

* Lower-priority interrupts are **blocked**
* Certain operations (like paging or waiting) **are not allowed**

Example:

* A network interrupt occurs → CPU raises IRQL to that device's level → interrupt handler runs.

So:

➡ **IRQL = OS priority level used by Windows to control interrupt handling**

---

## Key Difference

| Feature | IRQ                         | IRQL                                   |
| ------- | --------------------------- | -------------------------------------- |
| Type    | Hardware concept            | Software/kernel concept                |
| Used by | CPU + hardware              | Windows kernel                         |
| Purpose | Signal device needs service | Control interrupt priority and masking |
| Example | IRQ1 (keyboard)             | DISPATCH_LEVEL                         |

---

✅ **Relationship:**
When a device triggers an **IRQ**, the Windows kernel handles it by **raising the CPU to a corresponding IRQL** so the interrupt handler can run safely.

---

💡 Quick mental model:

* **IRQ → “Hardware yelling at the CPU.”**
* **IRQL → “Windows deciding what priority that yelling gets.”**

---

These are where IRQL becomes really important.

* **DIRQL vs IRQL**
* **Why “IRQL_NOT_LESS_OR_EQUAL” BSOD happens**
* **How spinlocks require DISPATCH_LEVEL**

**Modern Twist!**

* High-performance systems avoid interrupts entirely sometimes using polling (NAPI in Linux) or interrupt moderation because interrupts can become a bottleneck at 10–100 Gbps networking speeds.

**Interesting Fact!**

The actual CPU stack changes during an interrupt

Which looks like this:

```
RIP
CS
RFLAGS
RSP
SS
```

The CPU pushes these automatically before jumping to the ISR. It’s one of the coolest parts of CPU architecture.

---

**Hardware Block Diagram (IRQ Path)**

```
+------------------+
|  Hardware Device |
|  (NIC, Disk,    |
|   Keyboard)      |
+---------+--------+
          |
          |  IRQ line (electrical signal)
          |
          v
+----------------------+
| Interrupt Controller |
|  PIC / IO-APIC       |
|                      |
|  - collects IRQs     |
|  - prioritizes them  |
|  - routes to CPU     |
+----------+-----------+
           |
           | Interrupt signal
           v
+----------------------+
|      CPU             |
|  Local APIC / INTR   |
|                      |
| CPU pauses execution |
| and enters interrupt |
| handling routine     |
+----------+-----------+
           |
           v
+----------------------+
| Interrupt Vector     |
| Table Lookup (IDT)   |
+----------+-----------+
           |
           v
+----------------------+
| Interrupt Handler    |
| (OS / Driver ISR)    |
+----------------------+
```

**Circuit-Style View (How IRQ is Physically Triggered)**

```
              +5V
               |
               |
             [Pull-up]
               |
               +---------------------- IRQ Line --------------------+
               |                                                     |
               |                                                     |
       +-------+-------+                                     +-------+-------+
       |   Device      |                                     | Interrupt     |
       |   Controller  |                                     | Controller    |
       |               |                                     | (PIC/APIC)    |
       |   Transistor  |                                     |               |
       |     Switch    |                                     |  IRQ Input    |
       +-------+-------+                                     +-------+-------+
               |
               |
              GND
```

**Multiple Devices Sharing Interrupt Logic**

Real systems have many devices:

```
 Device A ----\
 Device B -----\ 
 Device C ------->  IO APIC  ---->  CPU
 Device D -----/
 Device E ----/
```

**Full Interrupt Path (Hardware → OS)**

```
+------------------+
|  Hardware Device |
|  (NIC, Disk,     |
|   Keyboard)      |
+---------+--------+
          |
          |  IRQ signal (electrical)
          v
+----------------------+
| Interrupt Controller |
|   IO-APIC / PIC      |
| - Collects IRQs      |
| - Prioritizes        |
| - Routes to CPU      |
+----------+-----------+
           |
           | Interrupt message
           v
+----------------------+
|  CPU Local APIC      |
|  (inside CPU)        |
+----------+-----------+
           |
           v
+======================+
|   CPU Interrupt      |
|   Entry Point        |
+======================+

        ↓ OS takes over ↓

+----------------------+
| Interrupt Descriptor |
| Table Lookup (IDT)   |
|  (find handler)      |
+----------+-----------+
           |
           v
+----------------------+
| ISR                  |
| Interrupt Service    |
| Routine (Driver)     |
| Runs at DIRQL        |
+----------+-----------+
           |
           v
+----------------------+
| DPC Queue            |
| Deferred Procedure   |
| Call                 |
| Runs at              |
| DISPATCH_LEVEL       |
+----------+-----------+
           |
           v
+----------------------+
| Kernel / Driver Work |
| Normal processing    |
| PASSIVE_LEVEL        |
+----------+-----------+
           |
           v
+----------------------+
| User Application     |
| Thread awakened     |
| (I/O completed)      |
+----------------------+
```

**Full Interrupt Path — Modern CPU Architecture → OS**

```
                HARDWARE (Motherboard / CPU)

+----------------------+
|  PCIe Device         |
|  (NIC / NVMe / GPU)  |
|                      |
| Device event occurs  |
+----------+-----------+
           |
           | Interrupt (MSI / MSI-X or IRQ)
           v
+----------------------+
|   IO APIC            |
|  Interrupt Controller|
|                      |
| - Collect interrupts |
| - Assign priority    |
| - Route to CPU core  |
+----------+-----------+
           |
           | Interrupt message
           v
+----------------------+
|  Local APIC          |
|  (inside CPU core)   |
|                      |
| Decides which core   |
| handles interrupt    |
+----------+-----------+
           |
           v
+======================+
|  CPU Interrupt Entry |
|  Context switch      |
+======================+

                OPERATING SYSTEM (Kernel)

           |
           v
+----------------------+
| Interrupt Descriptor |
| Table (IDT) Lookup   |
|                      |
| CPU finds ISR entry  |
+----------+-----------+
           |
           v
+----------------------+
| ISR                  |
| Interrupt Service    |
| Routine (Driver)     |
|                      |
| Runs at DIRQL        |
| Acknowledge device   |
+----------+-----------+
           |
           v
+----------------------+
| DPC Queue            |
| Deferred Procedure   |
| Call                 |
|                      |
| Runs at              |
| DISPATCH_LEVEL       |
+----------+-----------+
           |
           v
+----------------------+
| Kernel / Driver Work |
|                      |
| Runs at              |
| PASSIVE_LEVEL        |
+----------+-----------+
           |
           v
+----------------------+
| User Application     |
| Thread receives data |
| (I/O completed)      |
+----------------------+
```

