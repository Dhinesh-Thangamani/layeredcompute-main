# Data Transfer Between Non-SMM and SMM Mode

## Overview

System Management Mode (SMM) executes in a **physically isolated address space called SMRAM**, which is inaccessible from any non-SMM context (Ring 3, Ring 0, or even Ring -1 hypervisors). This isolation is what gives SMM its security guarantees, but it also creates a fundamental problem:

> How does non-SMM software (the OS kernel, a UEFI driver, a runtime service caller) hand data to an SMI handler — and receive results back — when the two sides cannot see each other's memory directly?

The answer is a small, well-defined set of **cross-privilege-boundary communication mechanisms**. Each mechanism uses a shared piece of memory (reachable from non-SMM) or CPU state (registers in the SMM save state area) as a *mailbox*, and uses a **software SMI** as the doorbell that wakes SMM up to read it.

This page describes why the communication is needed, the two canonical methods used by every modern UEFI platform, and other less-common variants.

---

## Why Non-SMM ↔ SMM Data Transfer Is Needed

SMM owns a number of resources and services that the OS and UEFI runtime environment cannot touch directly. Whenever non-SMM code needs one of those services, it has to ask SMM to do the work on its behalf. Typical reasons include:

* **SPI flash access** — The SPI controller is locked down after boot so that only SMM can write to it. `SetVariable()`, capsule updates, and BIOS updates all need to cross into SMM.
* **Authenticated UEFI variables** — Variables with `EFI_VARIABLE_AUTHENTICATED_WRITE_ACCESS` or `EFI_VARIABLE_TIME_BASED_AUTHENTICATED_WRITE_ACCESS` are validated and stored by SMM-resident code.
* **TPM / security coprocessor mediation** — Some platforms funnel TPM command submission through SMM for integrity reasons.
* **Platform configuration** — Reading or writing protected chipset registers, Intel ME / AMD PSP mailboxes, or locked-down MMIO ranges.
* **RAS and error logging** — Machine-check banks, PCIe AER logs, and memory error records are often harvested in SMM and exposed to the OS.
* **Capsule / firmware update staging** — The OS hands a capsule image to firmware; SMM validates and applies it.
* **Legacy emulation hooks** — USB legacy, PS/2 emulation, and similar quirks.

In every case, the caller must:

1. Place **input data** somewhere SMM can read it.
2. Trigger an **SMI** so SMM actually runs.
3. Let SMM write **output data** back to a location the caller can read.
4. Resume execution after `RSM`.

The methods below are the practical realizations of that pattern.

---

## Methods of Data Communication

### Method 1 — Software SMI with NVS / Runtime Data Buffer

This is the **classic, low-level mechanism** and is still the foundation of most vendor-specific SMI services. It predates the UEFI PI SMM specification and is used whenever a piece of OS-level code (ACPI, a BIOS utility, a Windows/Linux driver) needs to talk to SMM without the full UEFI PI infrastructure being available.

#### How it works

1. During boot, the firmware allocates a buffer in **ACPI NVS memory** (`EfiACPIMemoryNVS`) or **Runtime Services Data** (`EfiRuntimeServicesData`).
    * These memory types are preserved across `ExitBootServices()` and are reported to the OS so the OS will not reclaim them.
    * The physical address of the buffer is published — commonly through an **ACPI OperationRegion** in the DSDT/SSDT, a **SMBIOS OEM table**, or a fixed NVS pointer in a UEFI variable.
2. The caller (OS driver, ACPI ASL, or a UEFI module) writes its **command code** and **input parameters** into the buffer at the agreed-upon offsets.
3. The caller writes a specific **command value to I/O port 0xB2** (the APM Control Port, APMC). This generates a **software SMI**.
4. The CPU enters SMM. The SwSmi dispatcher inspects the value written to 0xB2 (and sometimes 0xB3 for a sub-function) and dispatches to the registered handler.
5. The SMM handler:
    * Reads the NVS / runtime buffer.
    * **Validates** every field (length, pointers, ranges — see [Security Considerations](#security-considerations)).
    * Performs the requested work.
    * Writes the result (status code + output data) back into the same buffer.
6. `RSM` returns to the caller, which reads the result from the buffer.

#### Typical layout of the shared buffer

```c
typedef struct {
    UINT32  Signature;      // e.g. 'SMIC'
    UINT32  Command;        // Sub-function selector
    UINT32  Status;         // Filled in by SMM on return
    UINT32  DataSize;       // Size of the payload that follows
    UINT8   Data[256];      // In/out payload
} OEM_SMI_NVS_BUFFER;
```

#### ACPI-driven example (ASL sketch)

```asl
OperationRegion (SMIB, SystemMemory, 0x7F000000, 0x100) // NVS buffer
Field (SMIB, DWordAcc, NoLock, Preserve) {
    CMD,  32,
    STAT, 32,
    DAT0, 32,
    DAT1, 32
}
OperationRegion (APMP, SystemIO, 0xB2, 2)
Field (APMP, ByteAcc, NoLock, Preserve) {
    APMC, 8,
    APMS, 8
}

Method (DOIT, 1) {
    Store (0x1234, CMD)       // Write command
    Store (Arg0,  DAT0)       // Write parameter
    Store (0xA0,  APMC)       // Trigger SW SMI
    Return (STAT)             // Read status filled in by SMM
}
```

#### Characteristics

| Property | Value |
| --- | --- |
| Trigger | Write to I/O port 0xB2 (APMC) |
| Transport | Pre-allocated NVS / Runtime buffer at a known physical address |
| Consumers | ACPI ASL, legacy OS drivers, pre-UEFI utilities |
| Spec | Vendor-defined / de-facto standard |
| Typical use | Thermal control, BIOS OEM services, legacy capsule |

#### Strengths

* Works from any context that can execute an I/O port write — including 16-bit code, ACPI ASL, and non-UEFI OSes.
* Zero infrastructure cost; needs only a buffer and an SMI handler.
* Extremely fast — a single OUT instruction.

#### Weaknesses

* **No standard schema**. Every OEM rolls its own command table and buffer layout, so portability is poor.
* **Fixed buffer size** set at boot — hard to change without reallocating.
* **Trust boundary is fragile**. The buffer lives in OS-writable memory, so SMM must treat every byte as hostile input and validate obsessively.
* Port 0xB2 is a **global resource** — all software SMIs collide there, forcing sub-function multiplexing.

---

### Method 2 — SMM Communicate Service (`EFI_MM_COMMUNICATION_PROTOCOL`)

This is the **UEFI-standardized mechanism** defined by the Platform Initialization (PI) specification, volume 4. It replaces ad-hoc OEM SMI buffers with a typed, discoverable, handler-addressed RPC framework and is what modern UEFI drivers and OSes should use.

It is exposed in two forms:

* Boot-time / DXE: `EFI_MM_COMMUNICATION_PROTOCOL` (formerly `EFI_SMM_COMMUNICATION_PROTOCOL`).
* Runtime: a **configuration table** (`EFI_MM_COMMUNICATION_TABLE` / `gEfiSmmCommunicationTableGuid`) pointing to a runtime-safe communicate buffer, so that OS-present UEFI runtime services (e.g. `SetVariable`) can still reach SMM after `ExitBootServices`.

#### How it works

1. Firmware allocates a **communication buffer** in runtime memory and publishes its address via the MM communication configuration table. The caller can also allocate its own buffer from `EfiRuntimeServicesData`.
2. The caller fills in an `EFI_MM_COMMUNICATE_HEADER`:

    ```c
    typedef struct {
        EFI_GUID  HeaderGuid;      // Identifies the target SMM handler
        UINTN     MessageLength;   // Size of Data[] that follows
        UINT8     Data[ANYSIZE];   // Handler-specific payload
    } EFI_MM_COMMUNICATE_HEADER;
    ```

    * `HeaderGuid` is the GUID the target handler registered itself under via `gMmst->MmiHandlerRegister()` / `SmiHandlerRegister()`.
    * `Data[]` is the payload — its format is defined by the handler whose GUID is in `HeaderGuid`.
3. The caller invokes `Communicate()`:

    ```c
    Status = MmCommunication->Communicate (
               MmCommunication,   // Protocol instance
               CommBuffer,        // Pointer to the header above
               &CommBufferSize    // In/out size
               );
    ```

4. The protocol implementation writes the buffer's physical address into a well-known location, then **triggers a software SMI** (the transport is still port 0xB2 under the hood on x86 — the communicate service is a layer *on top* of SW SMI, not a replacement for the hardware mechanism).
5. The SMM Core receives the SMI, walks the registered MMI handlers, and dispatches to the one whose GUID matches `HeaderGuid`. The handler signature is:

    ```c
    EFI_STATUS
    (EFIAPI *EFI_MM_HANDLER_ENTRY_POINT) (
        IN     EFI_HANDLE  DispatchHandle,
        IN     CONST VOID  *Context      OPTIONAL,
        IN OUT VOID        *CommBuffer   OPTIONAL,
        IN OUT UINTN       *CommBufferSize OPTIONAL
    );
    ```

6. The handler validates the buffer, performs the work, and writes the response in place. `Communicate()` returns and the caller reads the result.

#### Runtime variant (post-`ExitBootServices`)

The DXE protocol is no longer callable once boot services exit. Instead, firmware publishes `gEfiSmmCommunicationTableGuid` in the UEFI system configuration table, whose structure is:

```c
typedef struct {
    EFI_PHYSICAL_ADDRESS  SmmCommunicationBuffer;  // Pre-reserved runtime buffer
    UINT64                NumberOfEntries;
    EFI_MM_COMMUNICATE_HEADER_DESCRIPTOR Entries[];
} EFI_MM_COMMUNICATION_TABLE;
```

UEFI runtime services such as **Variable Services** use this table internally: `SetVariable()` after ExitBootServices packages the request into an `EFI_MM_COMMUNICATE_HEADER` with the SMM variable handler's GUID, drops it in the runtime communicate buffer, and triggers the SW SMI. The SMM-resident variable driver performs the authenticated write to SPI flash and returns status.

#### Characteristics

| Property | Value |
| --- | --- |
| Trigger | SW SMI (still port 0xB2), wrapped by the protocol |
| Transport | `EFI_MM_COMMUNICATE_HEADER` in runtime memory |
| Routing | GUID-based — handler chosen by `HeaderGuid` |
| Spec | UEFI PI Spec, Volume 4 (MM) |
| Typical use | UEFI variable services, capsule update, SMM RAS, OS-to-SMM RPC |

#### Strengths

* **Standard and portable** across firmware vendors.
* **Typed, GUID-addressed dispatch** — no fighting over port 0xB2 sub-functions.
* **Discoverable** via protocol database (boot) or configuration table (runtime).
* Scales to many SMM services without a central command-number registry.

#### Weaknesses

* Still requires strict input validation in SMM — the comm buffer is OS-writable memory.
* Slightly more overhead than raw port 0xB2 due to dispatch/lookup.
* Requires UEFI PI MM infrastructure; not available to legacy non-UEFI OSes.

---

### Method 3 — Other / Less-Common Mechanisms

Beyond the two canonical methods, several other transports are used in specific situations. They are either lower-level building blocks or special-purpose variants.

#### 3a. Register-based parameter passing via the SMM Save State

When an SMI is taken, the CPU saves GPRs, segment registers, RIP, RFLAGS, and more into the **SMM Save State Map** in SMRAM. The SMI handler is allowed to read — and in many implementations modify — this save state.

This means the caller can pass small amounts of data by putting them in registers (RAX, RBX, RCX, RDX…) immediately before the OUT to 0xB2. The SMI handler reads those register images from the save state.

* Used heavily by Intel **EFI\_SMM\_CPU\_PROTOCOL** (`ReadSaveState` / `WriteSaveState`).
* Good for tiny requests (a command code, a handle, a status).
* Limited to register-sized payloads — anything larger still needs a buffer.
* Used internally by UEFI variable services: the SW SMI value in RAX is the command, while the buffer address is read from a published pointer.

#### 3b. MSR / CPUID-based side channels

Some chipset- or CPU-specific flows use model-specific registers or special CPUID leaves to hand short parameters across the SMM boundary (e.g., Intel BIOS Guard, some debug and RAS paths). These are vendor-specific and not part of any portable API.

#### 3c. Hardware mailboxes (PCH / ME / PSP)

Certain services are implemented as hardware mailboxes backed by platform firmware (e.g. Intel ME HECI, AMD PSP mailbox, PCH SMBus controllers). SMM may act as a forwarder: non-SMM software calls SMM via Method 1 or Method 2, and SMM then marshals the real request into the hardware mailbox. From the OS's point of view the transport is still a SW SMI; the mailbox is just the back-end.

#### 3d. `EFI_MM_COMMUNICATION2_PROTOCOL`

A newer revision of the communicate protocol that accepts both a physical and a virtual address for the comm buffer, simplifying callers that run after `SetVirtualAddressMap()`. Semantically identical to Method 2 — just a nicer calling convention.

#### 3e. Periodic / synchronous SMI with pre-agreed memory

For telemetry-style flows (e.g., a periodic chipset SMI polling the ACPI NVS region for pending work), the non-SMM side just writes to a flag in NVS and lets the next periodic SMI discover it. No explicit trigger is needed. This is a polled variant of Method 1.

#### 3f. Hardware-triggered SMI carrying implicit data

In flows like thermal, GPIO, or PCIe error SMIs, the "data" is the **hardware state itself** — status bits in chipset registers — not a software-prepared buffer. The SMI handler reads the state, acts, and may publish a result to the OS via ACPI notify or a log in NVS. This is technically bidirectional communication, even though nothing was "handed in" through a buffer.

---

## Comparison Summary

| Aspect | Method 1: SW SMI + NVS/Runtime Buffer | Method 2: SMM Communicate | Method 3: Save-State / MSR / HW Mailbox |
| --- | --- | --- | --- |
| Standardized? | No — OEM-specific | Yes — UEFI PI Vol. 4 | No — platform-specific |
| Trigger | `OUT 0xB2, cmd` | `Communicate()` wrapping a SW SMI | Varies (OUT, WRMSR, MMIO doorbell) |
| Routing | Command value at port 0xB2/0xB3 | GUID in `EFI_MM_COMMUNICATE_HEADER` | Register contents / MSR / chipset state |
| Buffer type | `EfiACPIMemoryNVS` or `EfiRuntimeServicesData` | `EfiRuntimeServicesData` via MM comm table | Save state area / registers / HW buffer |
| Callable from legacy OS / ASL | Yes | No (needs UEFI PI) | Partially |
| Typical payload size | Up to buffer size (KB-range) | Up to comm buffer size | Tens of bytes |
| Discoverability | Ad-hoc (ACPI OpRegion, SMBIOS) | Protocol DB or config table | None |
| Best for | Legacy, OEM services, ACPI-to-SMM | UEFI variable, capsule, modern RPC | Tiny control paths, HW offload |

---

## Security Considerations

Regardless of which method is used, **every byte coming from non-SMM is hostile** as far as SMM is concerned. Common failure modes that have produced real CVEs:

* **Unvalidated pointers**: The comm buffer contains a pointer into OS memory; SMM dereferences it without checking it lies outside SMRAM, allowing an attacker to trick SMM into overwriting its own code — the classic **SMM call-out / confused-deputy** bug.
* **TOCTOU races**: SMM checks a length field, then re-reads it later. Because the buffer is OS-writable while SMM runs on other cores (or between SMIs), the length can change. Fix: copy the entire request into SMRAM first, then validate.
* **Integer overflow in length/offset fields** causing out-of-bounds reads or writes.
* **Missing origin checks** on the comm buffer address — the buffer must be verified to lie *entirely outside SMRAM* and in expected memory types.

Mitigations every SMI handler should implement:

1. Copy the request into SMRAM-local storage before validating.
2. Verify `CommBuffer` and `CommBuffer + CommBufferSize` fall outside SMRAM and inside allowed memory types (use `SmmIsBufferOutsideSmmValid()` in EDK II).
3. Reject pointers embedded in the payload unless they are bounded and re-validated.
4. Use hardware features (SMM\_Code\_Chk\_En, SMRR, SMM page tables with NX) to contain handler bugs.
5. Keep the handler small and auditable — every byte of SMM code runs at Ring -2.

---

## Conclusion

Non-SMM and SMM communicate through a very small set of patterns, all of which reduce to: **"shared memory + software SMI doorbell."** The two production-relevant methods are the legacy **SW SMI with NVS / runtime buffer** — flexible and usable from any environment including ACPI and legacy OSes — and the standardized **SMM Communicate service** — typed, GUID-dispatched, and the foundation of modern UEFI runtime services like authenticated variable writes and capsule updates. A handful of lower-level variants (save-state registers, MSR side channels, hardware mailboxes, MM Communicate v2, periodic SMI) fill in the remaining gaps, usually as building blocks rather than direct OS-to-SMM APIs.

The common thread across all of them is the trust model: non-SMM writes data into a buffer it controls, SMM is summoned, SMM treats the buffer as adversarial input, does the work, and writes a response in place. Getting that validation right is the single most important thing an SMM driver does — which is why so many historical SMM vulnerabilities trace back to sloppy handling of exactly these communication buffers.
