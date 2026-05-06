# **Windows IDT and Interrupts – Complete Overview**

---

## **1️⃣ IDT (Interrupt Descriptor Table)**

* **Definition**: A CPU-level table containing **256 entries** (vectors 0–255) that **point to interrupt service routines (ISRs)**.
* **Purpose**: When an interrupt occurs, the CPU uses the **vector number as an index into the IDT** to jump to the correct handler.

### Key points:

| Property          | Details                                                                   |
| ----------------- | ------------------------------------------------------------------------- |
| Number of entries | 256 (fixed)                                                               |
| Entry content     | Address of ISR or dispatcher routine                                      |
| Vector numbers    | 0–255; some reserved for CPU exceptions, some for hardware interrupts     |
| Static vs dynamic | Table size is **static**, but handler addresses may be updated at runtime |

---

### **Example: CPU exceptions in IDT**

| Vector | Handler                                  |
| ------ | ---------------------------------------- |
| 0x00   | Divide by zero → `nt!KiDivideErrorFault` |
| 0x03   | Breakpoint → `nt!KiBreakpointTrap`       |
| 0x0E   | Page fault → `nt!KiPageFault`            |

Hardware interrupts (e.g., keyboard, timer) are mapped to vectors usually **0x20–0x2F**.

---

### **Diagram 1: IDT Structure**

```
IDT (Interrupt Descriptor Table) - 256 entries

Vector 0x00 ─► nt!KiDivideErrorFault
Vector 0x01 ─► nt!KiDebugTrapOrFault
Vector 0x0E ─► nt!KiPageFault
Vector 0x20 ─► KiInterruptDispatch ──► ISR for Timer, Keyboard, etc.
Vector 0x21 ─► KiInterruptDispatch ──► ISR for Keyboard or shared devices
...
Vector 0xFF ─► Reserved
```

* **Each vector = index into IDT**
* **Dispatcher routine** often handles multiple ISRs

---

## **2️⃣ Interrupt Vector**

* **Definition**: The number used to **index into the IDT**.
* Historical reason: called “vector” because it **points the CPU to the handler address** (like an arrow).
* Modern usage: CPU reads **vector → looks up handler in IDT → jumps to ISR**.

**Analogy**: Room number in a hotel (vector) points to the staff (ISR) that handles requests.

---

## **3️⃣ Multiple ISRs per vector**

* Sometimes **one IDT entry points to a dispatcher** that handles **multiple ISRs**:

  * Example: Two devices share **IRQ1**, both handled by the dispatcher.
* WinDbg may show **more than one ISR for a vector**, meaning **the dispatcher calls multiple registered ISRs**.

**Diagram 2: Multiple ISRs per Vector**

```
Vector 0x21 ─► KiInterruptDispatch
                  │
       ┌──────────┴───────────┐
       │                      │
  ISR_DeviceA             ISR_DeviceB
```

* **IDT entry itself = fixed**
* **Dispatcher internally calls all registered ISRs**

---

## **4️⃣ _KINTERRUPT Object**

* **Definition**: Windows kernel structure representing an **interrupt object**
* Contains **everything Windows needs to know about an interrupt**: ISR, vector, IRQL, sharing info, etc.

**Key fields:**

| Field                   | Purpose                                       |
| ----------------------- | --------------------------------------------- |
| `ServiceRoutine`        | ISR for the interrupt                         |
| `MessageServiceRoutine` | For MSI/MSI-X devices                         |
| `Vector`                | Interrupt vector number                       |
| `Irql`                  | Interrupt Request Level (metadata)            |
| `SynchronizeIrql`       | IRQL used for synchronizing shared interrupts |
| `DispatchAddress`       | Dispatcher routine (`KiInterruptDispatch`)    |
| `ShareVector`           | Indicates vector sharing                      |
| `ActiveCount`           | Active instances of interrupt                 |
| `Connected`             | Whether ISR is connected                      |

---

### **Diagram 3: How _KINTERRUPT Fits in**

```
CPU Interrupt
      │
      ▼
   Vector # ──► IDT Entry ──► KiInterruptDispatch (Dispatcher)
                                  │
                       +----------+----------+
                       │                     │
               _KINTERRUPT (DeviceA)   _KINTERRUPT (DeviceB)
                       │                     │
               ServiceRoutine          ServiceRoutine
```

* `_KINTERRUPT` does **not execute**; it **stores metadata** for the dispatcher.

---

## **5️⃣ IRQL (Interrupt Request Level)**

* `_KINTERRUPT.Irql` = preferred IRQL for the interrupt.
* **Modern Hyper-V VMs** (MSI interrupts) often show **Irql = 0**:

  * The dispatcher dynamically raises IRQL at runtime
  * `_KINTERRUPT.Irql` is mostly **metadata**, not the actual runtime level

**Practical lab note**: Seeing 0 in `_KINTERRUPT` in Hyper-V is normal. Actual IRQL during ISR execution = `DISPATCH_LEVEL` or higher.

---

### **Diagram 4: IRQL Flow**

```
Interrupt fires
      │
      ▼
IDT entry ─► Dispatcher (KiInterruptDispatch)
      │
      ▼
Raise IRQL dynamically to DISPATCH_LEVEL
      │
Call ISR (_KINTERRUPT.ServiceRoutine)
      │
Lower IRQL after ISR completes
```

---

## **6️⃣ Summary Table**

| Concept          | Static / Dynamic   | Notes                                                   |
| ---------------- | ------------------ | ------------------------------------------------------- |
| IDT size         | Static (256)       | Entries fixed; handler addresses may change             |
| Interrupt vector | Fixed              | Index into IDT; points to ISR or dispatcher             |
| Multiple ISRs    | Dynamic            | Dispatcher may call multiple registered ISRs            |
| _KINTERRUPT      | Dynamic            | Stores ISR, vector, IRQL, sharing info; doesn’t execute |
| IRQL             | Dynamic at runtime | `_KINTERRUPT.Irql` often 0 in MSI/Hyper-V               |

---

✅ **Key Lab Takeaways**

1. `!idt` shows **all IDT entries with their handler addresses**
2. Some vectors have **multiple ISRs** because Windows uses a **shared dispatcher**
3. `_KINTERRUPT` objects store **interrupt metadata**
4. `_KINTERRUPT.Irql` in Hyper-V = 0 → dispatcher raises IRQL dynamically

---

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/9731ef83-6ec2-41bc-9ffd-89dd669ad6ff" />

---

`_kinterrupt` in the ReactOS repo is (in practice) referring to the Windows kernel’s **interrupt object type**: `struct _KINTERRUPT` (often written as `_KINTERRUPT`, and used via pointers like `PKINTERRUPT` / `struct _KINTERRUPT*`).

You can see the definition here:

```c name=sdk/include/ndk/ketypes.h url=https://github.com/reactos/reactos/blob/c9e73b6a77c9f1745d2c6fb7e9121d48a4a6ac3c/sdk/include/ndk/ketypes.h#L1037-L1115
typedef struct _KINTERRUPT
{
    CSHORT Type;
    CSHORT Size;
    LIST_ENTRY InterruptListEntry;
    PKSERVICE_ROUTINE ServiceRoutine;
#if (NTDDI_VERSION >= NTDDI_LONGHORN)
    PKSERVICE_ROUTINE MessageServiceRoutine;
    ULONG MessageIndex;
#endif
    PVOID ServiceContext;
    KSPIN_LOCK SpinLock;
    ULONG TickCount;
    PKSPIN_LOCK ActualLock;
    PKINTERRUPT_ROUTINE DispatchAddress;
    ULONG Vector;
    KIRQL Irql;
    KIRQL SynchronizeIrql;
    BOOLEAN FloatingSave;
    BOOLEAN Connected;
    CCHAR Number;
    BOOLEAN ShareVector;
    KINTERRUPT_MODE Mode;
#if (NTDDI_VERSION >= NTDDI_LONGHORN)
    KINTERRUPT_POLARITY Polarity;
#endif
    ULONG ServiceCount;
    ULONG DispatchCount;
#if (NTDDI_VERSION >= NTDDI_LONGHORN)
    ULONGLONG Rsvd1;
#endif
#ifdef _M_AMD64
    PKTRAP_FRAME TrapFrame;
    PVOID Reserved;
#endif
    ULONG DispatchCode[DISPATCH_LENGTH];
} KINTERRUPT;
```

### What it represents
- A `KINTERRUPT` instance represents a **registered hardware interrupt** in the kernel: vector/IRQL, ISR (`ServiceRoutine`), sharing mode, stats/counters, and synchronization fields (spin lock, etc.).
- Drivers commonly deal with it indirectly via APIs like `IoConnectInterrupt` / `IoConnectInterruptEx`, passing an ISR and context.

Example usage in ReactOS code (ISR signature uses `struct _KINTERRUPT*`):

```c name=win32ss/drivers/videoprt/interrupt.c url=https://github.com/reactos/reactos/blob/c9e73b6a77c9f1745d2c6fb7e9121d48a4a6ac3c/win32ss/drivers/videoprt/interrupt.c#L32-L47
BOOLEAN NTAPI
IntVideoPortInterruptRoutine(
   IN struct _KINTERRUPT *Interrupt,
   IN PVOID ServiceContext)
{
   PVIDEO_PORT_DEVICE_EXTENSION DeviceExtension = ServiceContext;

   ASSERT(DeviceExtension->DriverExtension->InitializationData.HwInterrupt != NULL);

   return DeviceExtension->DriverExtension->InitializationData.HwInterrupt(
      &DeviceExtension->MiniPortDeviceExtension);
}
```

### Note about search completeness
This information is based on an open-source project call ReactOS. To browse more occurrences in GitHub’s UI, use:

```text name=github-search.txt
https://github.com/reactos/reactos/search?q=_kinterrupt&type=code
```

