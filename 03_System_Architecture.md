---
tags: #parasite #phase-03 #architecture #hld #system-design
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 03: System Architecture — High-Level Design & Component Interaction

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 03 |
| Title | System Architecture (HLD) |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[01_Research_and_Validation]], [[02_Threat_Modeling]] |

---

## 1. Executive Summary

This document defines the high-level system architecture for the PARASITE project. It translates the findings from the research phase and the security requirements from the threat model into a concrete technical blueprint for development. The architecture is designed to meet the project's core goals: extreme efficiency (1.2KB binary), robust security (defense-in-depth), and broad portability across ARM, RISC-V, and Xtensa platforms.

The proposed architecture is a layered, modular design centered on the three core components: Sentinel, Guardian, and Reporter. These components are designed to run in a privileged, MPU-protected memory region, isolated from the main application firmware. This isolation is the cornerstone of the architecture, ensuring that PARASITE can maintain its integrity even if the host application is compromised.

Key architectural decisions include:
1.  **A Layered In-Firmware Model:** PARASITE is not an operating system but a co-resident agent. It is linked with the main application but executes in a separate, privileged security context enforced by hardware.
2.  **Hardware Abstraction Layer (HAL):** A minimal, standardized HAL is defined to ensure the core logic of PARASITE remains portable across different microcontroller families.
3.  **Event-Driven, State-Machine-Based Logic:** Each component is implemented as a finite state machine, which is highly efficient in terms of code size and predictable in execution. Communication between components is achieved via atomic, lock-free message queues.
4.  **API-less Kernel Entry:** To maximize security, the host application has no direct API access to privileged PARASITE functions. Instead, interaction is mediated through hardware exceptions (e.g., MPU faults), which act as a non-bypassable entry point into the PARASITE kernel.

This document details the design of each component, the data flows between them, the memory layout, the cryptographic architecture, and the key interfaces. It concludes with a series of Architecture Decision Records (ADRs) to formally document and justify critical design choices. This HLD serves as the definitive guide for the firmware, hardware, and backend development teams in subsequent phases.

---

## 2. Architecture Principles

- **Security First:** Every design decision is evaluated first through the lens of security. The system is designed to fail-secure, meaning that in the event of an internal error, it will default to the most secure state (e.g., locking down the device) rather than an insecure one.
- **Minimalism and Efficiency:** Every byte of code and every CPU cycle is accounted for. The architecture avoids dynamic memory allocation, complex data structures, and any feature that would compromise the strict size and performance budget.
- **Portability by Abstraction:** The core logic is written in pure, platform-agnostic Rust. All hardware-specific interactions are routed through a minimal Hardware Abstraction Layer (HAL) that must be implemented for each new target platform.
- **Zero-Trust Execution:** The PARASITE core treats the host application firmware as untrusted. It continuously monitors the application's behavior and memory space without relying on any services or guarantees from it.

---

## 3. High-Level Architecture

### 3.1 Layered Architecture View
This diagram illustrates the full system stack, from the physical hardware to the cloud backend.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 4: CLOUD BACKEND (VERIFIER)                               │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                 │
│ │  API GW / LB│ │Verifier Svc │ │  Intel DB   │                 │
│ └─────────────┘ └─────────────┘ └─────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
       ▲                                 (TLS 1.3 Mutual Auth)
       │
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 3: HOST FIRMWARE (NETWORK STACK)                          │
│ (Provides raw TCP/IP socket for Reporter)                       │
└─────────────────────────────────────────────────────────────────┘
       ▲
       │ (PARASITE Core invokes HAL calls for network I/O)
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 2: PARASITE CORE (MPU/PMP PROTECTED, PRIVILEGED)          │
│ ┌───────────┐   Event   ┌───────────┐   Alert   ┌───────────┐   │
│ │ SENTINEL  ├──────────►│ GUARDIAN  ├──────────►│ REPORTER  │   │
│ └───────────┘   Queue   └───────────┘   Queue   └───────────┘   │
└─────────────────────────────────────────────────────────────────┘
       ▲
       │ (Hardware Faults, Timer Interrupts, HAL calls)
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 1: HARDWARE ABSTRACTION LAYER (HAL)                       │
│ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐         │
│ │ MPU/PMP   │ │  Crypto   │ │  Flash    │ │  Network  │         │
│ │  Config   │ │  Engine   │ │  Driver   │ │  Driver   │         │
│ └───────────┘ └───────────┘ └───────────┘ └───────────┘         │
└─────────────────────────────────────────────────────────────────┘
       ▲
       │ (Direct Register Access)
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 0: HARDWARE                                               │
│ (MCU, Flash, RAM, MPU/PMP, Crypto Accelerator)                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Memory Layout
The memory map is critical to enforcing isolation. The linker script will be configured to place these sections at specific addresses.

```ascii
+-------------------------+ 0x080FFFFF
|      (Unused)           |
+-------------------------+
|   Application Data      | (RW, Unprivileged)
+-------------------------+
|   Application Code      | (RX, Unprivileged)
+-------------------------+ 0x08020000
|   PARASITE Data         | (RW, Privileged)
+-------------------------+
|   PARASITE Code         | (RX, Privileged)
+-------------------------+
|   Secure Bootloader     | (RX, Privileged)
+-------------------------+ 0x08000000 (Flash Start)

+-------------------------+ 0x2003FFFF
|   Quarantine Zone       | (No Access)
+-------------------------+
|   Application Stack/Heap| (RW, Unprivileged)
+-------------------------+
|   PARASITE Stack        | (RW, Privileged)
+-------------------------+ 0x20000000 (SRAM Start)
```

---

## 4. Component Design

### 4.1 SENTINEL Component
The Sentinel is a background task, driven by a low-frequency timer interrupt (e.g., every 10ms), that performs entropy scanning of application memory.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ SENTINEL COMPONENT DESIGN                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INPUT: Timer Interrupt, Dynamic Scan Requests                  │
│  OUTPUT: `EntropyAlert` event to Guardian's queue               │
│                                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ State Machine:                                            │   │
│ │  ┌──────┐   Timer   ┌──────────┐   Scan   ┌──────────┐    │   │
│ │  │ IDLE │──────────►│ SCANNING │─────────►│ ANALYZING│    │   │
│ │  └──────┘           └────┬─────┘          └────┬─────┘    │   │
│ │      ▲                   │                    │           │   │
│ │      │ Scan              │ Window             │ Chi²      │   │
│ │      │ Complete          │ Data               │ Result    │   │
│ │      └───────────────────┘                    │           │   │
│ │                          │                    ▼           │   │
│ │                          │            ┌──────────┐        │   │
│ │                          └───────────►│THRESHOLD │        │   │
│ │                                       │ CHECK    ├─[No]───┘   │
│ │                                       └─────┬────┘            │
│ │                                             │ [Yes]           │
│ │                                             ▼                 │
│ │                                       ┌──────────┐            │
│ │                                       │  QUEUE   │            │
│ │                                       │  ALERT   │            │
│ │                                       └──────────┘            │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│ Data Structures:                                                │
│  - `ScanQueue`: A circular buffer of memory regions to scan.    │
│  - `EntropyAlert`: { address: u32, length: u32, score: u32 }    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 GUARDIAN Component
The Guardian is a reactive component. It is idle until it receives an event from Sentinel or a hardware fault from the CPU.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ GUARDIAN COMPONENT DESIGN                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INPUT: `EntropyAlert` event, Memory Management Fault Exception │
│  OUTPUT: MPU reconfiguration, `ThreatReport` to Reporter's queue│
│                                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Event Loop:                                               │   │
│ │  1. Dequeue `EntropyAlert` from Sentinel's queue.         │   │
│ │  2. If no alert, sleep.                                   │   │
│ │  3. If alert:                                             │   │
│ │     a. Find an available MPU region.                      │   │
│ │     b. Configure MPU region to cover the alert address.   │   │
│ │     c. Set region permissions to "No Access".             │   │
│ │     d. Enable the MPU region.                             │   │
│ │     e. Create `ThreatReport` with IOC details.            │   │
│ │     f. Enqueue `ThreatReport` for the Reporter.           │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ Fault Handler (`MemManage_Handler`):                      │   │
│ │  1. CPU triggers fault on access violation.               │   │
│ │  2. Handler identifies faulting address and PC.           │   │
│ │  3. Create `ThreatReport` with advanced context.          │   │
│ │  4. Enqueue for Reporter.                                 │   │
│ │  5. Terminate faulting thread or reset device.            │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 REPORTER Component
The Reporter is a background task that manages the secure transmission of threat intelligence.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ REPORTER COMPONENT DESIGN                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INPUT: `ThreatReport` from Guardian's queue                    │
│  OUTPUT: TLS 1.3 packets via Network HAL                        │
│                                                                 │
│ ┌───────────────────────────────────────────────────────────┐   │
│ │ State Machine:                                            │   │
│ │  ┌──────┐ Dequeue  ┌──────────┐  Format   ┌──────────┐    │   │
│ │  │ IDLE │───[Yes]─►│PENDING   ├─────────► │PREPARING │    │   │
│ │  └──────┘          │ REPORT   │           │ TLS      │    │   │
│ │     ▲  │           └──────────┘           └────┬─────┘    │   │
│ │     │  └─[No]──┐                              │           │   │
│ │     │          │                              ▼           │   │
│ │     │          │                        ┌──────────┐      │   │
│ │     │          │                        │CONNECTING│      │   │
│ │     └──────────┼─── Transmit OK         └────┬─────┘      │   │
│ │                │                              │           │   │
│ │                │                              ▼           │   │
│ │                │                        ┌──────────┐      │   │
│ │                └────────────────────────┤TRANSMIT- │      │   │
│ │                   Transmit Fail         │TING      │      │   │
│ │                                         └──────────┘      │   │
│ └───────────────────────────────────────────────────────────┘   │
│                                                                 │
│ Key Operations:                                                 │
│  - Serialize `ThreatReport` into a compact format (e.g., nanopb).│
│  - Establish mTLS session using device key and server cert.     │
│  - Encrypt serialized report with ChaCha20-Poly1305.            │
│  - Transmit via Network HAL, with retry logic.                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Interface Design

### 5.1 Internal APIs (Component-to-Component)
Communication between PARASITE components is asynchronous, using lock-free, single-producer, single-consumer (SPSC) queues.

- `sentinel_queue_alert(alert: EntropyAlert)` -> `Result<(), Error>`
- `guardian_dequeue_alert()` -> `Option<EntropyAlert>`
- `guardian_queue_report(report: ThreatReport)` -> `Result<(), Error>`
- `reporter_dequeue_report()` -> `Option<ThreatReport>`

### 5.2 Hardware Abstraction Layer (HAL) API
This is the contract that must be fulfilled for each new platform. It's a set of C-ABI compatible functions that the core PARASITE Rust code will call.

A Rust definition of the HAL trait PARASITE expects:
```rust
/// The PARASITE Hardware Abstraction Layer (HAL) trait.
/// A concrete implementation of this trait must be provided for each target platform.
pub trait ParasiteHal {
    /// MPU/PMP Control
    fn mpu_init(&self, config: &MpuConfig);
    fn mpu_quarantine_region(&self, region_id: u8, addr: u32, len: u32) -> Result<(), HalError>;

    /// Cryptography
    fn get_device_id_key(&self) -> [u8; 32]; // Gets unique device private key
    fn aes_256_gcm_encrypt(&self, plaintext: &[u8], iv: &[u8]) -> Vec<u8>;

    /// Networking
    fn net_tcp_connect(&self, host: &str, port: u16) -> Result<SocketHandle, HalError>;
    fn net_tcp_write(&self, handle: SocketHandle, data: &[u8]) -> Result<usize, HalError>;
    fn net_tcp_read(&self, handle: SocketHandle, buffer: &mut [u8]) -> Result<usize, HalError>;
    
    /// Timers
    fn timer_start_periodic(&self, interval_ms: u32, callback: fn());
}
```

---

## 6. Security & Cryptographic Architecture

### 6.1 Key Hierarchy
The security of the entire reporting mechanism relies on a robust key hierarchy, rooted in hardware.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ KEY HIERARCHY                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ ┌───────────────────┐                                           │
│ │ HARDWARE ROOT KEY │ (HRK) - From PUF or eFuse                 │
│ │ (Device Unique)   │                                           │
│ └─────────┬─────────┘                                           │
│           │                                                     │
│           │ KDF (BLAKE2s)                                       │
│           ▼                                                     │
│ ┌───────────────────┐                                           │
│ │ DEVICE ID KEY     │ (DIK) - X25519 Private Key                │
│ │ (Used for mTLS)   │ Stored in Secure Element or encrypted flash │
│ └─────────┬─────────┘                                           │
│           │                                                     │
│           │ ECDH Key Exchange with Verifier Backend             │
│           ▼                                                     │
│ ┌───────────────────┐                                           │
│ │  SESSION KEY      │ (Ephemeral) - ChaCha20 Symmetric Key      │
│ │  (Used for Report)│ Exists only in PARASITE's private RAM     │
│ └───────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Device Attestation
The initial "hello" message from the Reporter to the Verifier backend will serve as a basic attestation. It will be a data structure containing hashes of the device's firmware and PARASITE's own binary, signed by the Device Identity Key (DIK). This proves to the backend that the report is coming from a genuine device running unmodified, authentic code.

---

## 7. Architecture Decision Records (ADRs)

### ADR-001: Why Rust for Firmware?
- **Decision:** The PARASITE core firmware will be implemented exclusively in Rust.
- **Justification:** The primary reason is security. Rust's ownership model and borrow checker eliminate entire classes of memory safety vulnerabilities (e.g., buffer overflows, use-after-free) at compile time. This is paramount for a security agent. Secondly, its zero-cost abstractions and fine-grained control over memory layout are essential for meeting the strict 1.2KB code size budget.
- **Alternatives Considered:** C, C++. C is the de facto standard but is notoriously prone to memory errors. C++ is safer but its complex features (exceptions, RTTI) can lead to significant code bloat.

### ADR-002: Why MPU over TrustZone?
- **Decision:** PARASITE will use the standard ARM MPU (or RISC-V PMP) for isolation, rather than requiring ARM TrustZone.
- **Justification:** Portability and availability. The MPU is a standard feature on virtually all ARM Cortex-M3/4/7/33 MCUs, including low-cost variants. TrustZone is only available on higher-end Cortex-M33/35/55 devices, which would severely limit PARASITE's addressable market, especially for retrofitting legacy systems. While TrustZone provides a more robustly isolated environment, a well-configured MPU provides sufficient protection for our threat model.
- **Alternatives Considered:** ARM TrustZone. Rejected due to limited availability and higher complexity.

### ADR-003: Why a Custom HAL instead of an RTOS?
- **Decision:** PARASITE will not depend on a Real-Time Operating System (RTOS). It will interface with hardware via a minimal, custom HAL.
- **Justification:** Footprint and integration simplicity. Requiring an RTOS would add significant code size (20KB+) and complexity. It would also force the host application to use that specific RTOS. By using a HAL, PARASITE can be linked into any firmware project, from bare-metal schedulers to RTOS-based applications, making adoption far easier for OEMs.
- **Alternatives Considered:** FreeRTOS, Zephyr. Rejected due to code size overhead and integration complexity.
