**What is CR8 register in x64 Processors?**

The **CR8 register** in x64 processors is used to control the **interrupt priority threshold**. It is closely related to how modern CPUs interact with the **Local APIC** interrupt priority system.

---

# CR8 Register in x64 Processors

**CR8 = Task Priority Register (TPR)**

It determines **which hardware interrupts the CPU is allowed to accept based on priority**.

If an interrupt has a **priority lower than CR8**, the CPU **will not deliver it yet**.

---

# Where CR8 Fits in the Interrupt System

```id="g0zqth"
Device interrupt
      ↓
IO APIC
      ↓
Local APIC
      ↓
Compare interrupt priority with CR8
      ↓
If priority >= CR8 → deliver interrupt
If priority <  CR8 → delay interrupt
```

So CR8 effectively acts as an **interrupt priority filter**.

---

# Historical Context

In 32-bit x86:

```
APIC Task Priority Register (TPR)
```

was accessed via **memory-mapped APIC registers**.

In **x64 processors**, the CPU provides **CR8 as a fast way to access that priority value**.

So:

```
CR8 = fast CPU interface to APIC TPR
```

---

# CR8 Layout

CR8 is a **64-bit control register**, but only the **low 4 bits are used**.

```
63                      4 3      0
+------------------------+--------+
|        reserved        |  TPR   |
+------------------------+--------+
```

The value represents the **interrupt priority class**.

Example:

```
CR8 = 0  → accept all interrupts
CR8 = 2  → block low priority interrupts
CR8 = F  → block almost all interrupts
```

---

# How Operating Systems Use CR8

Operating systems raise CR8 when they want to **temporarily block lower-priority interrupts**.

Example flow:

```
Normal execution
CR8 = 0
```

Then kernel enters critical section:

```
Raise IRQL
↓
Write value to CR8
↓
Low priority interrupts blocked
```

So:

```
Windows IRQL ↔ CR8 value
```

---

# Example in Windows

When Windows raises **IRQL**, it often writes to CR8.

Example mapping (simplified):

```
Windows IRQL        CR8 value
--------------------------------
PASSIVE_LEVEL  →    0
APC_LEVEL      →    1
DISPATCH_LEVEL →    2
Device IRQL    →    3+
```

This ensures lower-priority interrupts cannot interrupt the current work.

---

# Accessing CR8 in Assembly

Reading CR8:

```asm
mov rax, cr8
```

Writing CR8:

```asm
mov cr8, rax
```

Only **kernel mode** code can do this.

---

# Important Note

CR8 only affects **external interrupts** delivered via the **APIC**.

It does **not affect**:

* exceptions
* NMIs
* SMIs
* faults like page faults

Those always interrupt the CPU.

---

# Clean Mental Model

```
CR8 = interrupt priority threshold
```

Or visually:

```id="x01ibq"
Interrupt arrives
      ↓
Check interrupt priority
      ↓
Compare with CR8
      ↓
Allowed? → CPU handles interrupt
Blocked? → interrupt delayed
```

---

💡 **Interesting fact:**
CR8 is actually one of the **least discussed but most critical registers** in Windows kernel interrupt handling. It is the **hardware mechanism behind IRQL on x64 systems**.

---


