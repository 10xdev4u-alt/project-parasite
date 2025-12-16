---
tags: #parasite #phase-09 #attestation #security #cryptography
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 09: Attestation Protocol — Remote Integrity Verification

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 09 |
| Title | Attestation Protocol |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[03_System_Architecture]], [[08_Firmware_Core_Development]] |

---

## 1. Executive Summary

This document specifies the design of the PARASITE Remote Attestation protocol. While the Reporter module is designed for asynchronous, one-way reporting of detected threats, the attestation protocol provides a mechanism for the backend Verifier to proactively challenge a device and receive a cryptographically signed proof of its current state. This is a critical feature for periodic health checks, post-incident forensics, and high-confidence asset management.

The primary goal of this protocol is to allow the Verifier to answer the question: "Is this device running the exact, authentic firmware that I expect it to be running, right now?" The protocol is designed to be lightweight, secure, and resilient to replay attacks.

Our design is a classic **Challenge-Response** protocol:
1.  **Challenge:** The Verifier backend generates a large, random number (a **Nonce**) and sends it to the device as a challenge.
2.  **Response:** The device, upon receiving the challenge, generates an "Attestation Quote." This data structure contains the received Nonce, measurements (SHA-256 hashes) of its own critical memory regions (bootloader, PARASITE core, application firmware), and other device state information.
3.  **Signing:** The device then signs this entire Attestation Quote with its unique, hardware-backed Device Identity Key (DIK).
4.  **Verification:** The device sends the signed Quote back to the Verifier. The Verifier checks the signature using the device's known public key, validates the Nonce to prevent replay attacks, and compares the memory measurements against a database of "golden" values for that device's firmware version.

A successful verification provides a strong guarantee of the device's integrity at that specific moment in time. This document details the protocol flow, data structures, cryptographic operations, and the implementation plan within the PARASITE firmware core, ensuring a robust and trustworthy attestation mechanism.

---

## 2. Protocol Goals & Threat Model

### 2.1 Goals
-   **Freshness:** The attestation must prove the device's state *at the time of the challenge*, not some previous state. The Nonce is critical for this.
-   **Authenticity:** The attestation must be undeniably from the specific device being challenged. The device's unique signing key ensures this.
-   **Integrity:** The attestation must provide a verifiable measurement of the device's software. Hashing the firmware in memory achieves this.
-   **Efficiency:** The protocol must be lightweight enough to run on a constrained device without significantly impacting its primary function.

### 2.2 Threat Model
This protocol is designed to mitigate the following specific threats from Phase 02:
-   **T-ATT-01 (Replay Attack):** An attacker records a valid attestation response and replays it later to hide a subsequent compromise. **Mitigation:** The server-generated, single-use Nonce makes replayed responses invalid.
-   **T-ATT-02 (Spoofing):** A malicious device attempts to provide an attestation for a different, legitimate device. **Mitigation:** The response is signed by the legitimate device's unique, hardware-backed key, which the malicious device does not possess.
-   **T-ATT-03 (Firmware Tampering):** An attacker modifies the firmware on the device. **Mitigation:** The memory measurement (hash) in the attestation quote will not match the "golden" value on the server, causing verification to fail.

---

## 3. Protocol Flow

The protocol is a simple, three-step process initiated by the Verifier.

```ascii
   VERIFIER (Backend)                      DEVICE (Running PARASITE)
        │                                       │
        │      1. Attestation Challenge         │
        │ ────────────────────────────────────► │
        │      { nonce: [u8; 32] }              │
        │                                       │
        │                                       │ Generate Quote
        │                                       │ Hash Memory
        │                                       │ Sign Quote
        │                                       │
        │      2. Attestation Response          │
        │ ◄──────────────────────────────────── │
        │ { quote: Quote, signature: [u8; 64] } │
        │                                       │
        │                                       │
        │ 3. Verify Signature                   │
        │    Verify Nonce                       │
        │    Compare Hashes                     │
        │                                       │
        ▼                                       ▼
  [VALID/INVALID]                             [IDLE]
```

---

## 4. Data Structures

### 4.1 The Attestation Quote
This is the core data structure, containing all the evidence to be signed. It is designed to be compact and unambiguous.

```rust
// attestation/mod.rs

/// The Attestation Quote contains all evidence of the device's state.
/// This structure is serialized, hashed, and then signed.
#[repr(C)]
pub struct AttestationQuote {
    /// A magic number identifying this as a PARASITE quote.
    pub magic: u32,
    /// Version of the quote format.
    pub version: u32,
    /// The unique, server-provided nonce from the challenge.
    pub nonce: [u8; 32],
    /// The SHA-256 hash of the bootloader region.
    pub bootloader_measurement: [u8; 32],
    /// The SHA-256 hash of the PARASITE core region.
    pub parasite_measurement: [u8; 32],
    /// The SHA-256 hash of the main application region.
    pub application_measurement: [u8; 32],
    /// The current security version counter from the device.
    pub security_version: u32,
    /// A bitfield representing the current state of the Guardian.
    pub guardian_state: u32,
}

impl AttestationQuote {
    pub const MAGIC: u32 = 0xATT357ED;
}
```

### 4.2 The Attestation Response
This is what the device sends back to the Verifier.

```rust
// attestation/mod.rs

/// The complete response sent back to the verifier.
pub struct AttestationResponse {
    /// The serialized quote data.
    pub quote: AttestationQuote,
    /// The ECDSA P-256 signature of the SHA-256 hash of the serialized quote.
    pub signature: [u8; 64],
}
```

---

## 5. Implementation Plan

The attestation logic will be implemented as a new module within the PARASITE core. It will be triggered by a specific request from the Verifier, likely delivered over the same TLS channel as the Reporter uses.

### 5.1 Generating the Quote
This is the most critical part of the device-side implementation.

```rust
// attestation/mod.rs

use crate::hal::ParasiteHal;
use sha2::{Sha256, Digest};

// Memory layout constants defined by the linker script
extern "C" {
    static _bootloader_start: u8;
    static _bootloader_end: u8;
    static _parasite_start: u8;
    static _parasite_end: u8;
    static _app_start: u8;
    static _app_end: u8;
}

/// Generates, signs, and returns a complete Attestation Response.
pub fn generate_attestation_response(
    hal: &dyn ParasiteHal,
    nonce: [u8; 32]
) -> Result<AttestationResponse, &'static str> {

    // 1. Create the quote with the nonce and metadata
    let mut quote = AttestationQuote {
        magic: AttestationQuote::MAGIC,
        version: 1,
        nonce,
        bootloader_measurement: [0; 32],
        parasite_measurement: [0; 32],
        application_measurement: [0; 32],
        security_version: hal.get_security_version(),
        guardian_state: hal.get_guardian_state(),
    };

    // 2. Perform the memory measurements. This is a CPU-intensive operation.
    // It must be done carefully to avoid impacting real-time tasks.
    quote.bootloader_measurement.copy_from_slice(
        &measure_memory_region(
            unsafe { &_bootloader_start as *const u8 as usize },
            unsafe { &_bootloader_end as *const u8 as usize },
        )
    );
    quote.parasite_measurement.copy_from_slice(
        &measure_memory_region(
            unsafe { &_parasite_start as *const u8 as usize },
            unsafe { &_parasite_end as *const u8 as usize },
        )
    );
    quote.application_measurement.copy_from_slice(
        &measure_memory_region(
            unsafe { &_app_start as *const u8 as usize },
            unsafe { &_app_end as *const u8 as usize },
        )
    );

    // 3. Serialize the quote into a byte array for signing.
    let quote_bytes = unsafe {
        core::slice::from_raw_parts(
            &quote as *const AttestationQuote as *const u8,
            core::mem::size_of::<AttestationQuote>(),
        )
    };

    // 4. Hash the serialized quote.
    let quote_hash = Sha256::digest(quote_bytes);

    // 5. Use the HAL to sign the hash with the Device Identity Key.
    // The HAL abstracts the interaction with the Secure Element or crypto engine.
    let signature = hal.sign_with_device_key(&quote_hash)?;

    Ok(AttestationResponse { quote, signature })
}

/// Hashes a given memory region and returns the SHA-256 digest.
fn measure_memory_region(start_addr: usize, end_addr: usize) -> [u8; 32] {
    let region_slice = unsafe {
        core::slice::from_raw_parts(start_addr as *const u8, end_addr - start_addr)
    };
    Sha256::digest(region_slice).into()
}
```

### 5.2 Verifier Backend Logic
The Verifier backend will perform the inverse operation:
1.  Receive the `AttestationResponse`.
2.  Look up the device's public key based on its client certificate identity.
3.  Serialize the `quote` portion of the response into bytes.
4.  Hash the serialized bytes using SHA-256 to get `quote_hash`.
5.  Use the device's public key to verify the `signature` against the `quote_hash`. If it fails, the response is fraudulent.
6.  Check if the `nonce` in the quote matches the one it sent in the challenge. If it doesn't, it's a replay attack.
7.  Look up the "golden" measurements for the device's reported firmware version from a database.
8.  Compare the `bootloader_measurement`, `parasite_measurement`, and `application_measurement` from the quote with the golden values.
9.  If all checks pass, the device is marked as "Trusted." If any check fails, an alert is raised for a security operator to investigate.

This protocol provides a high degree of confidence in the integrity of a device, serving as a crucial complement to the real-time, autonomous detection capabilities of the core PARASITE agent.
