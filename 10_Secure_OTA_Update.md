---
tags: #parasite #phase-10 #ota #suit #firmware #security
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 10: Secure OTA Update — Field Updatability & SUIT Manifest

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 10 |
| Title | Secure OTA Update |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[06_Bootloader_Design]], [[03_System_Architecture]] |

---

## 1. Executive Summary

This document specifies the architecture for the Secure Over-the-Air (OTA) update mechanism for the PARASITE ecosystem. The ability to securely update firmware in the field is not a convenience but a critical security requirement. It allows for the patching of newly discovered vulnerabilities, the deployment of updated threat intelligence heuristics to the PARASITE agent, and the rollout of new features. An insecure update mechanism can be a primary attack vector, allowing an adversary to replace authentic firmware with a malicious version.

Our design builds directly upon the dual-bank memory layout and cryptographic verification capabilities of the secure bootloader designed in Phase 06. The process is designed to be **atomic**, **fail-safe**, and **secure**. It ensures that a power failure or connectivity loss during an update will not "brick" the device, and that only authentic, signed firmware from the OEM can ever be installed.

To align with emerging industry best practices and ensure interoperability, our design adopts the **IETF SUIT (Software Updates for IoT) standard**. The SUIT manifest is a standardized format for describing the firmware update, its dependencies, and the steps required to install it. This manifest, which contains the hash of the firmware binary, is itself signed by the OEM. The device will download the manifest and the firmware image, but it is the MCUBoot bootloader that will ultimately parse the manifest and perform the critical verification and swap operations, acting as the trusted agent for the update. This separation of concerns between the application (which downloads the update) and the bootloader (which installs it) is central to the security of the design.

---

## 2. OTA Architecture & Process Flow

The architecture leverages the dual-bank flash layout and the MCUBoot bootloader.

### 2.1 High-Level Process

```ascii
  OEM / UPDATE SERVER                  DEVICE (Application)                  DEVICE (MCUBoot)
        │                                     │                              │
        │ 1. Sign Firmware &                  │                              │
        │    Generate SUIT Manifest           │                              │
        │                                     │                              │
        │ 2. Publish Update                   │                              │
        │ ───────────────────────────────────►│ 3. Detect & Download Update  │
        │ (Manifest + Firmware Image)         │    (To Inactive Flash Bank)  │
        │                                     │                              │
        │                                     │ 4. Stage Update & Reboot     │
        │                                     │ ────────────────────────────►│ 5. Detect Staged Update
        │                                     │                              │
        │                                     │                                     │    Parse SUIT Manifest
        │                                     │                                     │    Verify Signature
        │                                     │                                     │    Verify Image Hash
        │                                     │                                     │    Check Anti-Rollback
        │                                     │                                     │
        │                                     │                                     │ 6. Perform Swap
        │                                     │                                     │    (Mark new bank as active)
        │                                     │                                     │
        │                                     │                                     │ 7. Boot from New Bank
        │                                     │                                     │
        │                                     │ 8. Confirm Update (Optional)        │
        │ ◄───────────────────────────────────│    (App sends status to server)     │
        │                                     │                                     │
        ▼                                     ▼                                     ▼
```

### 2.2 Roles & Responsibilities
-   **Application Firmware:** Its only job is to manage the download. It receives the update package (manifest + binary) from the server and writes it into the inactive flash bank. It has no ability to verify or install the update itself. Its final action is to make a single call to the bootloader services to "request" an update on the next boot.
-   **MCUBoot Bootloader:** It is the trusted "update agent." On boot, it detects the update request, reads the SUIT manifest from the inactive bank, and performs all cryptographic checks. Only if every check passes will it swap the active bank. This ensures a compromised application cannot bypass the security checks.

---

## 3. The SUIT Manifest

Using the IETF SUIT standard (RFC 9019) provides a robust, interoperable, and well-vetted format for describing the update. The manifest is a CBOR (Concise Binary Object Representation) file.

### 3.1 Key Manifest Components
A SUIT manifest for PARASITE will contain the following key information:
-   **Component Identifier:** Specifies that this update is for the main application firmware on this specific hardware model.
-   **Version Number:** The new firmware's semantic version.
-   **Security Counter:** The new anti-rollback security version.
-   **Image Digest:** The SHA-256 hash of the firmware binary. This is the most critical piece of information.
-   **Installation Directive:** A command that tells the bootloader to write the image to the active slot.
-   **Authentication Wrapper:** The entire manifest is wrapped in a structure that contains the digital signature (ECDSA P-256) and the identifier of the public key required to verify it.

### 3.2 Example Manifest Structure (Conceptual)

```
[
    // Authentication Wrapper
    1, // Signature Algorithm: ECDSA P-256
    h'DEADBEEF...' // Signature
    
    // Manifest
    [
        1, // Manifest Version
        2, // Sequence Number
        
        // Common Sequence of Commands
        [
            // 1. Validate Image Integrity
            1, // SUIT Directive: Validate
            [
                // Image Digest
                1, h'A1B2C3...' // SHA-256 Hash of firmware.bin
            ],
            
            // 2. Write the image
            3, // SUIT Directive: Write
            [
                // Component Identifier, Destination, etc.
            ]
        ]
    ]
]
```

---

## 4. Implementation Details

### 4.1 Application-Side Logic
The application needs an "OTA agent" task responsible for managing the download.

```rust
// ota_agent/mod.rs

// Represents the state of the OTA process in the application
pub enum OtaState {
    Idle,
    Downloading,
    ReadyToStage,
}

pub struct OtaAgent<'a> {
    hal: &'a dyn ParasiteHal,
    state: OtaState,
    // ... other fields for managing HTTP download, etc.
}

impl<'a> OtaAgent<'a> {
    pub fn poll(&mut self) {
        match self.state {
            OtaState::Idle => {
                // Check for new update availability from the server
                if self.check_for_update() {
                    self.start_download();
                    self.state = OtaState::Downloading;
                }
            }
            OtaState::Downloading => {
                // Manage the download process, writing chunks to the
                // inactive flash bank via the HAL.
                if self.is_download_complete() {
                    self.state = OtaState::ReadyToStage;
                }
            }
            OtaState::ReadyToStage => {
                // The download is complete. The only thing left to do
                // is to tell the bootloader to take over, then reboot.
                
                // This HAL function writes a magic value to a shared memory
                // region that MCUBoot will inspect on startup.
                self.hal.bootloader_request_update();
                
                // Trigger a system reset
                self.hal.system_reset();
            }
        }
    }
    
    // ... helper functions ...
}
```

### 4.2 Bootloader-Side Logic (MCUBoot)
MCUBoot already provides most of the necessary logic. Our primary work is to ensure it is configured correctly.
-   **Signature Verification:** We will configure MCUBoot to use the `p256r1` curve for ECDSA and to expect the public key at a specific location in its own flash area.
-   **SUIT Parser:** We will need to enable and integrate a lightweight SUIT manifest parser into the MCUBoot build. This parser will be responsible for extracting the image hash and other directives from the manifest.
-   **Anti-Rollback:** We will configure MCUBoot to use a region of flash to store the security version counter, enabling its built-in anti-rollback protection.

### 4.3 Fail-Safety
The entire process is designed to be fail-safe.
-   **Power loss during download:** The application will be able to detect an incomplete download on next boot and restart it. The inactive bank contains junk data, but the active bank is untouched.
-   **Power loss during verification/swap:** MCUBoot's swap algorithm is designed to be atomic. It first verifies the new image completely. Only then does it change the metadata that marks the new bank as active. If power is lost during the swap itself, on next boot it will detect a corrupt state and can revert to the last known-good bank. The device will never be left in a non-bootable state.

---

## 5. Delta (Differential) Updates

While Version 1 of the PARASITE OTA mechanism will handle full firmware images, the architecture is designed to support delta updates in the future. This is a critical feature for reducing bandwidth consumption and airtime costs, especially for cellular-connected devices.

-   **Concept:** Instead of sending the full 256KB firmware image, the server would send a compact "patch" file that describes the binary differences between the old version and the new version.
-   **Process:**
    1.  The server generates the patch using a binary diffing tool (e.g., `bsdiff`).
    2.  The SUIT manifest is updated to include the hash of the patch file and a new directive: `ProcessPatch`.
    3.  The device downloads the manifest and the small patch file.
    4.  The bootloader, after verifying the manifest, reads the *old* firmware from the active bank, applies the patch to it in RAM, and writes the *reconstructed new* firmware to the inactive bank.
    5.  From this point, the process is identical: the bootloader hashes the newly reconstructed image and compares it to the hash in the manifest. If they match, the update is successful.

This provides the same level of security as a full update (as the final result is still cryptographically verified) but with a significant reduction in data transmission size. Support for this will be a key goal for Version 2.
