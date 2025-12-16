---
tags: #parasite #phase-00 #index #master
version: 1.1
status: Living Document
author: princetheprogrammer
---

# Phase 00: Index & Summary — PARASITE Project Master Index

## Document Metadata
| Field        | Value               |
| ------------ | ------------------- |
| Phase        | 00                  |
| Title        | Index & Summary     |
| Version      | 1.1                 |
| Last Updated | 2025-12-16          |
| Author       | princetheprogrammer |
| Status       | Living Document     |

---

## 1. Executive Summary

The integrity of India's Critical National Infrastructure (CNI) is foundational to its economic stability and national security. However, this foundation is increasingly threatened by a sophisticated class of cyber-attacks targeting the firmware of embedded systems. These devices—numbering in the hundreds of millions—form the operational backbone of the nation's power grids, telecommunications, transportation, and defense sectors. Current security paradigms, focused on network perimeter defense and traditional IT endpoints, are fundamentally blind to malicious code executing at the bare-metal level. This creates a critical, unmonitored vulnerability for Advanced Persistent Threats (APTs) to establish deep, persistent footholds, enabling catastrophic disruption, espionage, and sabotage.

The PARASITE (Polymorphic Adaptive Response And Sentinel for Infrastructure Threat Elimination) initiative is a direct response to this strategic threat. It proposes the development and deployment of a hyper-efficient, legally-grounded firmware sentinel, engineered to operate within the extreme resource constraints of embedded microcontrollers. With a target binary size of just 1.2KB, PARASITE can be seamlessly retrofitted into both legacy and modern devices. Its innovative three-layer architecture provides a complete "detect, contain, report" lifecycle. **Layer 1 (Sentinel)** employs a novel, real-time entropy analysis engine to detect the statistical anomalies characteristic of polymorphic malware, bypassing signature-based methods entirely. **Layer 2 (Guardian)** leverages on-chip hardware primitives like the Memory Protection Unit (MPU) to surgically isolate and contain detected threats in microseconds, ensuring device integrity and operational continuity. **Layer 3 (Reporter)** establishes a cryptographically secure channel to transmit actionable, anonymized threat intelligence to designated national security agencies, transforming compromised devices from liabilities into a distributed, national-scale sensor network.

PARASITE's key differentiators are its near-zero overhead, its platform-agnostic design spanning ARM, RISC-V, and Xtensa architectures, and its strict adherence to an ethical, defense-only operational mandate under the Indian IT Act. Unlike costly and cumbersome commercial alternatives, PARASITE is designed for ubiquitous, low-cost deployment. The expected outcome of this project is not merely a piece of software, but a paradigm shift in CNI protection: a scalable, sovereign capability that hardens India's infrastructure from the silicon up. This project will deliver a production-ready prototype, a comprehensive legal and deployment framework, and a clear path to empanelment with CERT-In, providing India with unprecedented visibility and control over its firmware threat landscape.

---

## 2. Project Vision & Mission

### 2.1 Vision Statement
To establish a new global standard in firmware security, creating a resilient, self-defending ecosystem of embedded devices across India's critical infrastructure that guarantees national sovereignty and the uninterrupted delivery of essential services.

### 2.2 Mission Statement
To engineer, validate, and deploy PARASITE, a hyper-efficient, legally-compliant, and ethically-grounded firmware sentinel that autonomously detects, contains, and reports advanced cyber threats in real-time, transforming vulnerable embedded systems into a cohesive national threat intelligence network.

### 2.3 Core Values
- **Security through Integrity:** We commit to the highest standards of security, focusing on the intrinsic integrity of device execution rather than relying on fallible signatures. Our work begins and ends with protecting the device.
- **Privacy by Design:** The system is architected from the ground up to ensure privacy. All data collection is minimal, strictly necessary for threat identification, and anonymized at the source before transmission.
- **Transparent Operations:** The project's methodology, operational parameters, and legal framework will be transparent to all stakeholders. Trust is a non-negotiable prerequisite for the deployment of security infrastructure.
- **Sovereign Capability:** PARASITE is an initiative to build indigenous, world-class security technology. This reduces dependency on foreign entities and ensures that the tools used to protect India's infrastructure are fully understood and controlled domestically.

### 2.4 Success Metrics

| Category | Metric | Target |
|----------|--------|--------|
| **Technical** | Detection Rate (Polymorphic Implants) | > 99.5% |
| | False Positive Rate (Operational) | < 0.01% |
| | CPU Overhead (Average) | < 0.1% |
| | Binary Size (ARMv7-M) | < 1,248 Bytes |
| | Containment Time (Median) | < 350 µs |
| **Deployment** | Platform Coverage | STM32, nRF52, ESP32, GD32V |
| | Pilot Program | Live deployment with one CNI partner |
| | Scalability Test | 1M+ virtual nodes in HIL simulation |
| **Impact** | CERT-In Empanelment | Achieved |
| | Public Threat Report | Published Annually |
| | Academic Contribution | 1+ Paper in a Tier-1 Conference |

---

## 3. Master Phase Overview

The project is divided into 16 interdependent phases, ensuring a structured progression from concept to a fully realized system.

```ascii
┌──────────────────────────────────────────────────────────────────────┐
│ PARASITE DEVELOPMENT ROADMAP                                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ [00] Index ──► [01] Research ──► [02] Threat Model ──► [03] Arch     │
│                                                                      │
│    │             │                │               │                  │
│    └─────────────┼────────────────┼───────────────┘                  │
│                  ▼                ▼               ▼                  │
│             [04] BOM         [05] Circuit     [06] Bootloader        │
│                  │                │               │                  │
│                  └────────────────┼───────────────┘                  │
│                                   ▼                                  │
│                              [07] HW Proto                           │
│                                   │                                  │
│                                   ▼                                  │
│ [11] Key Mgmt ◄── [08] FW Core ──► [09] Attestation ──► [12] Verifier│
│                  │                │                  │               │
│                  ▼                │                  │               │
│             [10] Secure OTA       │                  │               │
│                  │                │                  │               │
│                  └────────────────┴──────────────────┘               │
│                                   │                                  │
│                                   ▼                                  │
│                              [13] Integration ──► [14] Testing       │
│                                   │                 │                │
│                                   └─────────────────┼────────────────┘
│                                                     ▼                |
│                                             [15] Docs ──► [16] Pitch |
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Phase Deliverable Matrix

| Phase | Name | Duration (Days) | Dependencies | Key Deliverables | Risk |
|-------|------|-----------------|--------------|------------------|------|
| **00** | Index & Summary | 1 | - | Master Index, Roadmap, This Document | Low |
| **01** | Research & Validation | 3 | 00 | Threat Landscape Report, Prior Art Analysis, Feasibility Study | Low |
| **02** | Threat Modeling | 2 | 01 | STRIDE/DREAD Analysis, Attack Trees, Security Requirements | Medium |
| **03** | System Architecture | 3 | 02 | HLD, Component Interaction Diagrams, API Contracts | Medium |
| **04** | Bill of Materials (BOM) | 2 | 03 | Component Selection, Cost Analysis, Supply Chain Strategy | Low |
| **05** | Circuit Design | 3 | 04 | Schematics, Power Budget, Layout Specification | Medium |
| **06** | Bootloader Design | 3 | 03 | Secure Boot Specification, Anti-Rollback Mech, Recovery Flow | High |
| **07** | Hardware Prototype | 4 | 05 | Assembled PCBs, Functional Test Jig | High |
| **08** | Firmware Core Dev | 5 | 03, 06 | Sentinel, Guardian, Reporter Modules (Rust) | High |
| **09** | Attestation Protocol | 3 | 08 | Quote/Verify Logic, Nonce Generation, Message Formats | Medium |
| **10** | Secure OTA Update | 3 | 08 | SUIT Manifest Agent, Delta Update Logic, Rollback Plan | High |
| **11** | Key Management | 2 | 03 | Key Hierarchy Design, Secure Storage & Derivation Spec | High |
| **12** | Verifier Backend | 4 | 09 | Cloud Verifier Service, Intel DB Schema, Alerting Engine | Medium |
| **13** | System Integration | 3 | 07, 12 | End-to-End Testbed, Full System Functional Demo | High |
| **14** | Testing & Certification | 4 | 13 | Penetration Test Report, IEC 62443 Compliance Map | Medium |
| **15** | Troubleshooting Guide | 2 | 14 | FMEA Document, Debugging Playbook, Support Manual | Low |
| **16** | Final Packaging & Pitch | 2 | 13 | Pitch Deck, Final Report, Enclosure Design | Low |

---

## 5. Technology Stack Overview

### 5.1 Hardware Platforms
The choice of target architectures is strategic, covering the vast majority of CNI and IoT devices.

```ascii
        ┌──────────────────────────────┐
        │      TARGET ARCHITECTURES    │
        └──────────────┬───────────────┘
                       │
    ┌──────────────────┼──────────────────┐
    │                  │                  │
┌───▼───┐         ┌────▼────┐         ┌────▼────┐
│  ARM  │         │ Xtensa  │         │ RISC-V  │
└───┬───┘         └────┬────┘         └────┬────┘
    │                  │                   │
┌───┴───┐         ┌────┴────┐         ┌────┴────┐
│Cortex-M│        │ LX6/LX7 │         │ RV32IMC │
└───┬───┘         └────┬────┘         └────┬────┘
    │                   │                   │
┌───┴─────────┐ ┌───────┴───────┐ ┌────────┴────────┐
│• STM32L5/F4 │ │• ESP32        │ │• GD32VF103      │
│• nRF52/91   │ │• ESP32-S2/S3  │ │• CH32V series   │
│• LPC series │ │               │ │• ESP32-C3       │
└─────────────┘ └───────────────┘ └─────────────────┘
```

### 5.2 Software Stack
A layered, modular software architecture is employed for security, portability, and maintainability.

```ascii
┌─────────────────────────────────────────────────────────┐
│ LAYER 4: APPLICATION FIRMWARE (OEM/OWNER)               │
│ (Interacts with PARASITE via defined API)               │
├─────────────────────────────────────────────────────────┤
│ LAYER 3: PARASITE CORE (MPU/PMP PROTECTED)              │
│ ┌───────────┐   ┌───────────┐   ┌───────────┐           │
│ │ SENTINEL  │──▶│ GUARDIAN  │──▶│ REPORTER  │           │
│ └───────────┘   └───────────┘   └───────────┘           │
├─────────────────────────────────────────────────────────┤
│ LAYER 2: PLATFORM ABSTRACTION LAYER (PAL)               │
│ ┌───────────┐ ┌───────────┐ ┌───────────┐               │
│ │  MPU/PMP  │ │  CRYPTO   │ │  FLASH    │               │
│ │  DRIVER   │ │  DRIVER   │ │  DRIVER   │               │
│ └───────────┘ └───────────┘ └───────────┘               │
├─────────────────────────────────────────────────────────┤
│ LAYER 1: SECURE BOOTLOADER (e.g., MCUBoot)              │
│ (Verifies and launches Application + PARASITE)          │
├─────────────────────────────────────────────────────────┤
│ LAYER 0: HARDWARE                                       │
└─────────────────────────────────────────────────────────┘
```

### 5.3 Development Tools
- **Firmware:**
    - **Language:** Rust (Embedded). Chosen for its compile-time memory and thread safety guarantees, which eliminate entire classes of vulnerabilities, and its zero-cost abstractions, which are critical for achieving the 1.2KB target.
    - **IDE:** VS Code + `rust-analyzer`.
    - **Build System:** `cargo` with `lto="fat"`, `codegen-units=1`, and size optimization (`-Oz`).
- **Hardware:**
    - **EDA Tool:** KiCad.
    - **Debug Tools:** `probe-rs` and Segger J-Link.
- **CI/CD & Testing:**
    - **Automation:** GitHub Actions.
    - **HIL Testing:** `harness` (custom Rust HIL framework), `defmt` for high-performance logging.
- **Backend:**
    - **Language:** Go. Chosen for its excellent concurrency support, performance, and simple deployment story.
    - **Database:** TimescaleDB (on top of PostgreSQL) for efficient storage and querying of time-series threat data.
    - **Infrastructure:** Docker and Kubernetes on a sovereign cloud provider.

---

## 6. Team Structure & Responsibilities

### 6.1 Core Team Roles
```ascii
            ┌──────────────────┐
            │  Project Lead    │
            │ (Overall Vision) │
            └────────┬─────────┘
                     │
    ┌────────────────┼────────────────────┐
    │                │                    │
┌───▼────────────┐ ┌───▼────────────┐ ┌───▼──────────┐
│ Firmware Lead  │ │ Hardware Lead  │ │ Backend Lead │
│(Rust, Security)│ │(Schematic, PCB)│ │(Cloud, Infra)│
└────────┬───────┘ └──────┬─────────┘ └──────┬───────┘
         │                │                  │
    ┌────┴────┐        ┌──┴──┐            ┌──┴──┐
    │ FW Eng 1│        │     │            │     │
    └─────────┘        └─────┘            └─────┘
    ┌────┴────┐
    │ FW Eng 2│
    └─────────┘
```

### 6.2 RACI Matrix
| Phase | Responsible | Accountable | Consulted | Informed |
|-------|-------------|-------------|-----------|----------|
| 01-03 | Security Arch | Project Lead | FW, HW Leads | Team |
| 04-05 | HW Lead | Project Lead | FW Lead | Team |
| 06,08,10 | FW Lead | Project Lead | Security Arch | Team |
| 07 | HW Lead | Project Lead | FW Lead | Team |
| 09,12 | Backend Lead | Project Lead | FW Lead, Sec Arch | Team |
| 11 | Security Arch | Project Lead | FW, Backend Leads | Team |
| 13-14 | All Leads | Project Lead | - | Team |
| 15-16 | Project Lead | Project Lead | All Leads | - |

---

## 7. Timeline & Milestones

### 7.1 Gantt Chart
```ascii
Phase | Name                  | Wk 1    | Wk 2    | Wk 3    | Wk 4    | Wk 5    |
──────┼───────────────────────┼─────────┼─────────┼─────────┼─────────┼─────────┤
00-02 | Research & Arch       |███████▏ |         |         |         |         |
03-04 | HLD & BOM             |  ███████████▏ |         |         |         |
05-06 | Circuit & Bootloader  |         |█████████████▏|         |         |
07    | HW Prototype          |         |         |█████████▏|         |
08,11 | FW Core & Key Mgmt    |         |         |█████████████████████▏|
09-10 | Attestation & OTA     |         |         |         |█████████████▏|
12    | Verifier Backend      |         |         |         |█████████████████▏
13    | System Integration    |         |         |         |         |█████████▏
14    | Testing & Cert        |         |         |         |         |  █████████▏
15-16 | Docs & Pitch          |         |         |         |         |    ███████▏
```

### 7.2 Critical Path Analysis
The project's critical path is defined by the sequence of dependent tasks that determine the project's minimum duration. Any delay on this path directly translates to a delay in project completion.
**Critical Path:** `00 -> 01 -> 02 -> 03 -> 06 -> 08 -> 13 -> 14 -> 16`
This path underscores the dependency of all firmware development on the foundational architecture and bootloader design, and the final integration's reliance on a functional firmware core.

### 7.3 Key Milestones & Go/No-Go Gates
- **M1 (End of Wk 1): Architecture Sign-off.** *Gate: Is the HLD technically feasible and does it meet all security requirements derived from the threat model?*
- **M2 (End of Wk 2): Hardware BOM Freeze.** *Gate: Are all critical components available from secure supply chains with acceptable lead times and costs?*
- **M3 (End of Wk 3): Prototype Power-on.** *Gate: Does the custom hardware pass basic electrical and functional tests?*
- **M4 (End of Wk 4): Core Firmware Modules Complete.** *Gate: Do Sentinel, Guardian, and Reporter function correctly in isolation on the prototype hardware?*
- **M5 (End of Wk 5): End-to-End System Demo.** *Gate: Can a simulated threat be detected, contained, and reported through the entire system, from device to cloud backend?*

---

## 8. Risk Register Summary

| ID | Risk Description | P | I | Score | Mitigation Strategy |
|----|------------------|---|---|-------|---------------------|
| R01 | Binary size exceeds 1.2KB target. | 4 | 5 | 20 | Continuous size monitoring in CI/CD pipeline; nightly builds fail if size exceeds budget. Use of `twig` for code size profiling. |
| R02 | Hardware prototype fabrication/assembly delayed. | 3 | 5 | 15 | Dual-source vendors for PCB fab and assembly. Begin firmware development on commercial dev kits in parallel. |
| R03 | MPU/PMP containment logic introduces instability. | 3 | 5 | 15 | Develop extensive HIL test suite to fuzz MPU configurations and identify race conditions or unintended memory faults. |
| R04 | High false positive rate from Sentinel. | 4 | 3 | 12 | Implement a two-stage detection: initial anomaly detection followed by a short-term learning/baselining period on first boot to profile the host firmware. |
| R05 | Secure boot chain is compromised by a vulnerability in the bootloader itself. | 2 | 5 | 10 | Select a community-vetted, security-audited bootloader (MCUBoot). Commission a third-party code review for integration logic. |

---

## 9. Budget Overview

| Category | Description | Estimated Cost (INR) |
|----------|-------------|----------------------|
| **Hardware** | Prototype PCBs, components, assembly (10 units) | ₹ 1,50,000 |
| **Dev Kits** | ARM, Xtensa, RISC-V development boards | ₹ 50,000 |
| **Test Equipment** | Debug probes, logic analyzer, power monitor | ₹ 1,20,000 |
| **Software** | Cloud Hosting (1 year), specialized licenses | ₹ 80,000 |
| **Legal/Compliance**| Legal consultation on IT Act, CERT-In process | ₹ 1,00,000 |
| **Contingency** | 15% of total | ₹ 75,000 |
| **TOTAL** | | **₹ 5,75,000** |

---

## 10. Document Navigation

### 10.1 Quick Links
- [[01_Research_and_Validation]]
- [[02_Threat_Modeling]]
- [[03_System_Architecture]]
- [[04_Bill_of_Materials]]
- [[05_Circuit_Design]]
- [[06_Bootloader_Design]]
- [[07_Hardware_Prototype]]
- [[08_Firmware_Core_Development]]
- [[09_Attestation_Protocol]]
- [[10_Secure_OTA_Update]]
- [[11_Key_Management]]
- [[12_Verifier_Backend]]
- [[13_System_Integration]]
- [[14_Testing_and_Certification]]
- [[15_Troubleshooting_Guide]]
- [[16_Final_Packaging_and_Pitch]]

### 10.2 Cross-Reference Matrix
(R = Reads from, W = Writes to)
| Phase | 01 | 02 | 03 | 06 | 08 | 11 | 12 |
|-------|----|----|----|----|----|----|----|
| **01** | - | W | W | - | - | - | - |
| **02** | R | - | W | - | W | W | - |
| **03** | R | R | - | W | W | W | W |
| **06** | - | - | R | - | W | R | - |
| **08** | - | R | R | R | - | R | W |
| **11** | - | R | R | R | R | - | R |
| **12** | - | - | R | - | R | R | - |

---

## 11. Glossary & Acronyms
- **APT:** Advanced Persistent Threat
- **CNI:** Critical National Infrastructure
- **HLD:** High-Level Design
- **HIL:** Hardware-in-the-Loop
- **IOC:** Indicator of Compromise
- **MPU:** Memory Protection Unit
- **PAL:** Platform Abstraction Layer
- **PMP:** Physical Memory Protection (RISC-V)
- **SUIT:** Software Updates for IoT (IETF standard)

---

## 12. Revision History
| Version | Date       | Author              | Changes                                                                          |
| ------- | ---------- | ------------------- | -------------------------------------------------------------------------------- |
| 1.0     | 2025-12-16 | princetheprogrammer | Initial creation of the master index document.                                   |
| 1.1     | 2025-12-16 | princetheprogrammer | Reworked for professional tone, new naming, and higher detail per user feedback. |

---

## 13. Appendices

### A: File Naming Conventions
- Phase Documents: `XX_Name_in_Pascal_Case.md` (e.g., `01_Research_and_Validation.md`)
- Source Code: `snake_case.rs`
- Hardware Files: `parasite_mainboard_v1.kicad_sch`

### B: Obsidian Tag Taxonomy
- `#parasite`: Global project tag.
- `#phase-XX`: Tag for a specific phase.
- `#topic-<name>`: e.g., `#topic-security`, `#topic-rust`, `#topic-hardware`.
- `#status-<state>`: e.g., `#status-draft`, `#status-review`, `#status-complete`.
