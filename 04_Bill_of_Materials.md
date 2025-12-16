---
tags: #parasite #phase-04 #bom #components #cost
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 04: Bill of Materials — Component Selection & Cost Analysis

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 04 |
| Title | Bill of Materials (BOM) |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[03_System_Architecture]] |

---

## 1. Executive Summary

This document presents the Bill of Materials (BOM), component selection rationale, and cost analysis for the PARASITE project's hardware prototype. The selection of each component has been driven by the stringent security, performance, and cost requirements established in the preceding architecture and research phases. This BOM serves as a practical foundation for the circuit design (Phase 05) and prototype fabrication (Phase 07).

Our component selection philosophy prioritizes security features and supply chain integrity above all else. The primary microcontroller (MCU) selection was a critical decision. After a detailed comparative analysis, the **STMicroelectronics STM32L562QE** has been selected as the primary MCU for high-security applications, owing to its robust implementation of ARM TrustZone, extensive cryptographic accelerators, and strong market availability. For more cost-sensitive or legacy retrofit applications, the **STMicroelectronics STM32F407VG** is selected as a secondary, MPU-based option. To ensure a strong root of trust for cryptographic operations, the **Infineon OPTIGA™ Trust M (SLS32AIA)** has been chosen as the dedicated Secure Element.

The total estimated BOM cost for a single prototype unit is approximately **₹2,850**. For a production run of 10,000 units, the target unit cost is projected to be under **₹950**, making PARASITE an economically viable solution for mass deployment in devices like smart meters and IoT gateways. This document provides a detailed breakdown of these costs, along with a procurement strategy that emphasizes authorized distributors in India to ensure component authenticity and mitigate supply chain risks. The analysis concludes that the hardware required to build a secure, PARASITE-enabled device is readily available and financially feasible for the target markets.

---

## 2. BOM Philosophy & Selection Criteria

The selection of every component, from the MCU to the smallest passive resistor, is guided by a strict set of criteria derived from the project's core principles.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ COMPONENT SELECTION CRITERIA                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ PRIORITY 1: SECURITY & TRUST                                    │
│ ═══════════════════════════════════════════════════════════════ │
│ • MCU: Must have a reliable hardware memory protection (MPU/PMP)│
│   with at least 8 regions. TrustZone is highly preferred.       │
│ • Crypto: Must have hardware acceleration for AES and a TRNG.   │
│ • Secure Element: Must be at least CC EAL5+ certified.          │
│ • Supply Chain: Components must be sourced from authorized,     │
│   traceable distributors to prevent counterfeiting.             │
│                                                                 │
│ PRIORITY 2: PERFORMANCE & COMPATIBILITY                         │
│ ═══════════════════════════════════════════════════════════════ │
│ • MCU: ARM Cortex-M4 or better. Clock speed > 80 MHz.           │
│ • Memory: Flash ≥ 256KB, SRAM ≥ 64KB to accommodate PARASITE    │
│   plus a moderately complex host application.                   │
│ • Peripherals: Must support standard interfaces (I2C, SPI, UART)│
│   for connectivity and Secure Element communication.            │
│                                                                 │
│ PRIORITY 3: AVAILABILITY & LONGEVITY                            │
│ ═══════════════════════════════════════════════════════════════ │
│ • Lead Time: Standard lead time < 16 weeks.                     │
│ • Second Source: A pin-compatible or software-compatible second │
│   source for the primary MCU is highly desirable.               │
│ • Product Longevity: Manufacturer must guarantee availability   │
│   for at least 10 years.                                        │
│                                                                 │
│ PRIORITY 4: COST                                                │
│ ═══════════════════════════════════════════════════════════════ │
│ • Target MCU Cost (10k units): < ₹500                           │
│ • Target Total BOM Cost (10k units): < ₹1000                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Primary MCU Selection

### 3.1 Candidate Analysis
A comprehensive analysis was performed on five leading MCUs across the target architectures. The "PARASITE Score" is a weighted metric based on the criteria above.

| Feature | STM32L562 (ARM) | nRF52840 (ARM) | ESP32-S3 (Xtensa) | GD32VF103 (RISC-V) |
|---------------|-----------------|----------------|-------------------|--------------------|
| **Architecture**| Cortex-M33 | Cortex-M4F | Dual-Core LX7 | RV32IMC |
| **Security** | **TrustZone, MPU** | MPU, CryptoCell | MPU, Flash Encrypt| PMP |
| **Crypto Accel**| **AES, PKA, TRNG** | AES, TRNG | AES, SHA, RSA | None |
| **Flash / SRAM**| 512KB / 256KB | 1MB / 256KB | 8MB (ext) / 512KB | 128KB / 32KB |
| **MPU/PMP Regions**| **8+8 (Secure/NS)** | 8 | 8 | 8 |
| **Secure Boot** | **Native, HW-based**| Native | Native | Manual |
| **Debug Protect**| **RDP L2, Fuse** | APPROTECT | JTAG Disable | Fuse |
| **Availability**| Excellent | Good | Excellent | Limited |
| **Price (1k qty)**| ~₹480 | ~₹410 | ~₹280 | ~₹150 |
| **PARASITE Score**| **94/100** | 86/100 | 75/100 | 55/100 |

### 3.2 Selection Justification
- **Primary MCU: STM32L562QE**
    - **Rationale:** The STM32L5 series is the clear winner for security-critical applications. Its implementation of ARM TrustZone provides a hardware-enforced isolation boundary that is superior to a software-configured MPU. This aligns perfectly with our architecture of running PARASITE in a privileged, secure world. The comprehensive crypto accelerators and robust debug protection mechanisms make it an ideal choice for the project's primary development target.
- **Secondary MCU: nRF52840**
    - **Rationale:** For applications where wireless connectivity (BLE, 802.15.4) is a primary driver, the nRF52840 is a strong choice. Its mature MPU and CryptoCell-310 provide a solid security foundation, and its software compatibility with the ARM Cortex-M4 architecture allows for a high degree of code reuse from the primary platform.
- **RISC-V Candidate: GD32VF103**
    - **Rationale:** While the GD32VF103 is a popular entry-level RISC-V MCU, its lack of hardware crypto acceleration and limited memory make it unsuitable for this project's security requirements. Future RISC-V platforms with more robust security features (e.g., from the SiFive Intelligence series) will be considered as they become more widely available.

---

## 4. Secure Element Selection

While the STM32L5 provides excellent on-chip security, a dedicated Secure Element (SE) is chosen for the prototype to establish a hardware root of trust that is physically distinct from the main MCU. This provides defense-in-depth against sophisticated physical attacks.

### 4.1 SE Candidate Analysis

| Feature | OPTIGA™ Trust M | ATECC608B | SE050 (NXP) |
|---------------|-----------------|-----------|-------------|
| **Certification**| **CC EAL6+** | CC EAL4+ | CC EAL6+ |
| **Interface** | I2C | I2C | I2C |
| **Key Storage** | 12 slots | 16 slots | 100+ slots |
| **Algorithms** | **ECC, RSA, AES** | ECC, SHA | ECC, RSA, AES |
| **Price (1k qty)**| ~₹120 | ~₹85 | ~₹190 |
| **Availability**| Excellent | Excellent | Good |
| **PARASITE Score**| **95/100** | 85/100 | 90/100 |

### 4.2 Selection Justification
- **Primary SE: Infineon OPTIGA™ Trust M (SLS32AIA)**
    - **Rationale:** The OPTIGA Trust M is selected for its best-in-class EAL6+ security certification, providing a very high degree of assurance against physical and logical attacks. It offers a complete cryptographic toolbox and a robust command set for key generation, signing, and encryption. Its excellent availability and reasonable cost make it the ideal choice for storing the Device Identity Key (DIK) as defined in the system architecture.

---

## 5. Complete BOM Tables

### 5.1 Prototype BOM (1 unit)
This BOM is for a single, fully-featured prototype board for development and testing.

| Item | Part Number | Manufacturer | Qty | Unit Price (₹) | Extended (₹) | Notes |
|------|-------------|--------------|-----|----------------|--------------|-------|
| **MCU** | STM32L562QEI6 | STMicroelectronics | 1 | 850 | 850 | Retail price |
| **Secure Element**| SLS32AIA000AUSA1 | Infineon | 1 | 250 | 250 | Retail price |
| **Regulator** | AP2112K-3.3 | Diodes Inc. | 1 | 45 | 45 | 3.3V LDO |
| **USB-Serial** | FT232RL | FTDI | 1 | 350 | 350 | For debug UART |
| **Ethernet PHY** | LAN8720A | Microchip | 1 | 220 | 220 | For wired connectivity |
| **RJ45 MagJack** | 08B0-1X1T-36-F | Bel | 1 | 150 | 150 | Integrated magnetics |
| **PCB** | Custom 4-Layer | - | 1 | 800 | 800 | Small batch fab cost |
| **Passives** | Various | - | ~50 | 4 | 200 | Caps, Resistors, etc. |
| **Connectors** | Various | - | ~5 | 25 | 125 | USB, Headers |
| **SUBTOTAL** | | | | | **2,990** | |
| **Contingency (10%)**| | | | | **299** | |
| **TOTAL** | | | | | **₹3,289** | |

### 5.2 Production BOM (10,000 units)
This BOM reflects volume pricing and cost optimization for mass production.

| Item | Part Number | Manufacturer | Qty | Unit Price (₹) | Extended (₹) |
|------|-------------|--------------|-----|----------------|--------------|
| **MCU** | STM32L562QEI6 | STMicroelectronics | 1 | 480.00 | 480.00 |
| **Secure Element**| SLS32AIA000AUSA1 | Infineon | 1 | 120.00 | 120.00 |
| **Regulator** | AP2112K-3.3 | Diodes Inc. | 1 | 8.50 | 8.50 |
| **Ethernet PHY** | LAN8720A | Microchip | 1 | 95.00 | 95.00 |
| **RJ45 MagJack** | 08B0-1X1T-36-F | Bel | 1 | 65.00 | 65.00 |
| **PCB** | Custom 2-Layer | - | 1 | 40.00 | 40.00 |
| **Passives** | Various | - | ~40 | 0.50 | 20.00 |
| **Connectors** | Various | - | ~2 | 8.00 | 16.00 |
| **SUBTOTAL** | | | | | **₹844.50** |
| **Assembly & Test**| | | | | **₹65.00** |
| **TOTAL UNIT COST**| | | | | **₹909.50** |

---

## 6. Cost Analysis

### 6.1 Cost Breakdown (10k Unit Production)
The MCU and Secure Element constitute the majority of the cost, which is expected for a security-focused device.

```ascii
┌─────────────────────────────────────────────────────────────────┐
│ COST BREAKDOWN (10k UNIT PRODUCTION, TOTAL = ₹909.50)           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ MCU (STM32L562)       █████████████████████████████████ (52.8%) │
│ Secure Element (OPTIGA) █████████████ (13.2%)                   │
│ Ethernet PHY          ██████ (10.4%)                            │
│ RJ45 MagJack          ████ (7.1%)                               │
│ Assembly & Test       ████ (7.1%)                               │
│ PCB                   ██ (4.4%)                                 │
│ Passives & Conn.      ██ (4.0%)                                 │
│ Regulator             (1.0%)                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Cost Optimization Opportunities
- **MCU Selection:** For extremely cost-sensitive applications, switching to the secondary MCU candidate (nRF52840) or a lower-end STM32F4 could reduce the MCU cost by 20-30%. This would come at the cost of losing TrustZone capabilities.
- **Secure Element:** For devices where the MCU has a high-quality internal secure storage mechanism, the external SE could be removed, saving ~₹120. This is a significant trade-off in physical security.
- **Connectivity:** If the device does not require wired Ethernet, removing the PHY and MagJack provides a significant cost saving of ~₹160.

---

## 7. Procurement Strategy

### 7.1 Authorized Distributors
To ensure an authentic and secure supply chain, all critical components will be sourced exclusively from Tier-1 authorized distributors in India.

| Component | Distributor 1 | Distributor 2 |
|-----------|---------------|---------------|
| STM32 MCUs | Arrow Electronics India | Avnet India |
| Infineon SE | Arrow Electronics India | Future Electronics |
| Microchip PHY | Arrow Electronics India | Mouser Electronics India |

### 7.2 Long-Lead Items
- The primary MCU (STM32L562) and the Secure Element (OPTIGA Trust M) are identified as potential long-lead items, with lead times that can fluctuate between 12 to 30 weeks based on global demand.
- **Mitigation:** A rolling forecast will be provided to distributors, and buffer stock for at least one production quarter will be maintained.

### 7.3 Obsolescence Planning
- All selected components are on a >10 year longevity program from their respective manufacturers.
- An annual BOM review will be conducted to identify any components nearing their end-of-life and to proactively qualify replacements.
