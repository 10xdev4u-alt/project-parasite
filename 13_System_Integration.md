---
tags: #parasite #phase-13 #integration #testing #e2e
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 13: System Integration — End-to-End Flow & Testing

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 13 |
| Title | System Integration |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[07_Hardware_Prototype]], [[08_Firmware_Core_Development]], [[12_Verifier_Backend]] |

---

## 1. Executive Summary

This document outlines the strategy and execution plan for the system integration phase of the PARASITE project. This phase is the critical juncture where all previously designed and individually developed components—the hardware prototype, the secure bootloader, the core firmware agent, and the cloud backend—are brought together for the first time. The primary goal is to create a fully functional, end-to-end system and to verify that it operates as a cohesive whole, successfully executing the core "detect, contain, report" mission.

Our integration strategy is incremental, designed to isolate and resolve issues in a structured manner. It is divided into three stages:
1.  **Firmware Integration:** Merging the MCUBoot bootloader with the application firmware, which includes the PARASITE core (Sentinel, Guardian, Reporter). This stage focuses on ensuring the chain of trust is correctly established and the PARASITE agent is initialized properly.
2.  **Hardware/Firmware Integration:** Deploying the fully integrated firmware onto the physical hardware prototype from Phase 07. This stage focuses on validating the Hardware Abstraction Layer (HAL) and ensuring the firmware correctly interacts with the physical MCU, Secure Element, and peripherals.
3.  **Full System (End-to-End) Integration:** Connecting the validated hardware/firmware assembly to the live, containerized staging environment of the Verifier Backend. This final stage validates the entire communication pipeline, from on-device threat detection to the appearance of an alert in the backend database.

A comprehensive End-to-End (E2E) test plan is presented, centered on a "Red Team" scenario where a simulated malicious implant is introduced onto the device. The success criterion is the successful detection, containment, and reporting of this implant, verified at each stage of the system. This document provides the blueprint for transforming the individual project components into a single, demonstrable, and functional security ecosystem.

---

## 2. Integration Strategy & Staging

The integration will proceed in a logical order, building upon a stable foundation at each step.

### Stage 1: Firmware Integration (Host-based)
-   **Goal:** To link the bootloader and application/PARASITE firmware and verify the boot process in a simulated environment.
-   **Environment:** A host-based test environment using QEMU (or similar) to emulate the target MCU.
-   **Steps:**
    1.  Configure the MCUBoot build scripts to be aware of the application's memory layout.
    2.  Create a "dummy" application that links the PARASITE core library.
    3.  Sign the dummy application binary using the firmware signing key.
    4.  Create a single flash image containing MCUBoot and the signed application.
    5.  Load and run this image in QEMU.
-   **Success Criteria:** MCUBoot successfully verifies the application signature, boots the application, and the PARASITE core initializes and begins its idle loop (verified via debug logs).

### Stage 2: Hardware/Firmware Integration
-   **Goal:** To deploy the integrated firmware onto the physical prototype and validate the HAL.
-   **Environment:** A validated hardware prototype board (from Phase 07) connected to a debugger and logic analyzer.
-   **Steps:**
    1.  Flash the integrated firmware image from Stage 1 onto the prototype board via the debug interface.
    2.  Power cycle the board.
    3.  Verify via debugger that MCUBoot runs and successfully jumps to the application entry point.
    4.  Verify via debug logs that the PARASITE HAL correctly initializes all necessary peripherals (MPU, timers, etc.).
    5.  Manually trigger a test alert and verify that the Guardian correctly programs the MPU regions by reading back the MPU registers with the debugger.
-   **Success Criteria:** The full firmware stack boots successfully on physical hardware, and all HAL functions operate as expected.

### Stage 3: Full System End-to-End Integration
-   **Goal:** To connect the device to the cloud backend and verify the entire communication flow.
-   **Environment:** The validated hardware prototype, connected to the internet. A staging deployment of the Verifier Backend running in a Kubernetes cluster.
-   **Steps:**
    1.  Provision the device with its final Device Identity Certificate.
    2.  Configure the firmware with the URL of the staging backend.
    3.  Power on the device.
    4.  Trigger the E2E test scenario (see below).
-   **Success Criteria:** The threat report generated on the device is successfully received, processed, and stored by the backend, and is visible via a database query.

---

## 3. End-to-End (E2E) Test Plan: "Red Team One"

This test plan simulates a realistic attack scenario from start to finish.

### 3.1 Scenario
An attacker has gained initial access to the device and has managed to execute a small payload that attempts to write a "low-entropy" mock implant into an executable memory region.

### 3.2 Test Flow Diagram

```ascii
┌──────────────────┐   1. Write mock implant   ┌──────────────────┐
│ Test Harness     ├──────────────────────────►│ App Firmware     │
│ (Rust, on Host)  │   (via debug interface)   │ (on Device)      │
└──────────────────┘                           └────────┬─────────┘
                                                        │ 2. Implant is written
                                                        │    to RAM.
                                                        ▼
                                                 ┌──────────┐
                                                 │ SENTINEL │
                                                 └────────┬─┘
                                                        │ 3. Sentinel scan detects
                                                        │    anomalous write.
                                                        ▼
                                                 ┌──────────┐
                                                 │ GUARDIAN │
                                                 └────────┬─┘
                                                        │ 4. Guardian quarantines
                                                        │    the memory region.
                                                        │ 5. Guardian queues report.
                                                        ▼
                                                 ┌──────────┐
                                                 │ REPORTER │
                                                 └────────┬─┘
                                                        │ 6. Reporter encrypts and
                                                        │    sends report via TLS.
                                                        ▼
                                                 ┌──────────┐
                                                 │ VERIFIER │
                                                 │ BACKEND  │
                                                 └────────┬─┘
                                                        │ 7. Backend decrypts,
                                                        │    verifies, and stores.
                                                        ▼
                                                 ┌──────────┐
                                                 │  THREAT  │
                                                 │ DATABASE │
                                                 └──────────┘
```

### 3.3 Verification Steps

| Step | Action | Expected Outcome | Verification Method |
|------|--------|------------------|---------------------|
| 1 | The test harness writes the mock implant into the application's RAM. | The write succeeds. | Debugger memory view. |
| 2 | Wait for the next Sentinel scan cycle (e.g., 10ms). | Sentinel detects the anomaly. | `defmt` log output from the device shows a "High Entropy Detected" event. |
| 3 | The Guardian receives the alert. | The corresponding memory region is made "No Access" by the MPU. | `defmt` log shows "Quarantining region...". An attempted read of the region by the test harness via the debugger now fails. |
| 4 | The Reporter receives the threat report. | A TLS connection is established to the backend. | Wireshark/tcpdump capture shows a successful TLS handshake. |
| 5 | The Verifier Backend receives the report. | The report is successfully decrypted, authenticated, and stored. | Log output from the Intel Service pod in Kubernetes shows "Processing new threat report...". |
| 6 | Query the backend database. | A new entry exists in the `threat_events` table corresponding to the device and the event details. | `SELECT * FROM threat_events WHERE device_id = ?` returns the correct record. |

---

## 4. Test Harness Implementation (Conceptual)

To trigger the E2E test, we need a test harness that can control the device under test (DUT). This Rust-based harness will run on a host PC and use the `probe-rs` library to interact with the DUT via a debug probe.

```rust
// test_harness/main.rs

use probe_rs::{Probe, Permissions};
use std::time::Duration;

// The mock implant is designed to have low entropy to bypass
// the primary Chi-squared test, testing our secondary heuristics.
const MOCK_IMPLANT: &[u8] = b"\x90\x90\x90\x90\xDE\xAD\xBE\xEF"; // NOP sled + payload
const TARGET_ADDRESS: u32 = 0x2001_0000; // A known RAM address in the app region

fn main() -> Result<(), anyhow::Error> {
    // 1. Connect to the debug probe (e.g., J-Link)
    let probes = Probe::list_all();
    if probes.is_empty() {
        anyhow::bail!("No debug probes found");
    }
    let mut probe = probes[0].open()?;
    probe.attach("STM32L562QE", Permissions::default())?;
    let mut session = probe.attach_to_unspecified_core()?;
    let mut core = session.core(0)?;

    println!("-> Connected to device. Halting core.");
    core.halt(Duration::from_secs(1))?;

    // 2. Write the mock implant directly into the device's RAM
    println!("-> Writing mock implant to 0x{:08X}", TARGET_ADDRESS);
    core.write_8(&mut MOCK_IMPLANT, TARGET_ADDRESS)?;

    // 3. Resume the core to let PARASITE do its work
    println!("-> Resuming core. PARASITE should now be active.");
    core.run()?;

    println!("-> Implant injected. Now monitoring backend for results...");
    
    // 4. The rest of the test involves polling the backend API or DB
    // to verify that the report was received.
    // ...

    Ok(())
}
```
This harness provides a repeatable, automated way to simulate the initial compromise, allowing us to run the full E2E test as part of our continuous integration pipeline.
