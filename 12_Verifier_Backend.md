---
tags: #parasite #phase-12 #backend #cloud #go #architecture
version: 1.1
status: In Progress
author: princetheprogrammer
---

# Phase 12: Verifier Backend — Cloud Architecture & Services

## Document Metadata
| Field | Value |
|-------|-------|
| Phase | 12 |
| Title | Verifier Backend |
| Version | 1.1 |
| Last Updated | 2025-12-16 |
| Author | princetheprogrammer |
| Dependencies | [[09_Attestation_Protocol]], [[03_System_Architecture]] |

---

## 1. Executive Summary

This document specifies the architecture and design of the PARASITE Verifier Backend. This cloud-native system is the authoritative counterpart to the on-device firmware agent. It serves as the central hub for receiving threat intelligence, verifying device integrity through the attestation protocol, and providing a management interface for security operators. The backend must be designed to be highly scalable, resilient, and secure, capable of handling communication from millions of deployed devices.

Our architecture is based on a modern, microservices-oriented approach, designed for deployment on a sovereign cloud infrastructure using Docker and Kubernetes. The primary language for service development will be Go, chosen for its excellent performance, built-in concurrency, and suitability for network services.

The backend is composed of several key services:
-   **API Gateway:** A single, secure entry point for all device communication. It terminates TLS, authenticates devices using their client certificates, and routes requests to the appropriate backend service.
-   **Attestation Service:** Implements the server-side logic of the attestation protocol. It generates challenges (nonces), receives signed attestation responses, and performs the full cryptographic and logical verification against a database of known-good firmware measurements.
-   **Intel Service:** A dedicated endpoint for receiving and processing the asynchronous threat reports sent by the Reporter module on devices. It ingests, normalizes, and stores Indicators of Compromise (IOCs).
-   **Device Registry:** A database and service that manages device identities, their corresponding public keys, and their current trust status.
-   **Threat Intelligence Database:** A time-series database (TimescaleDB) optimized for storing and analyzing the vast amounts of threat data collected from the field.

This document details the technology stack, service interactions, API endpoints, and database schema. It provides a complete blueprint for building a robust, scalable, and secure cloud backend to support the PARASITE ecosystem at a national scale.

---

## 2. Technology Stack

| Component | Technology | Rationale |
|-----------|------------|-----------|
| **Language** | Go | High performance, excellent concurrency model (`goroutines`), strong standard library for networking, simple deployment (static binaries). |
| **API Framework** | Gin | Lightweight, high-performance HTTP web framework for Go. |
| **Database (Relational)** | PostgreSQL | Robust, open-source, and reliable for storing structured data like device identities and firmware metadata. |
| **Database (Time-Series)** | TimescaleDB | A PostgreSQL extension optimized for time-series data. Perfect for storing and analyzing threat intelligence events (IOCs). |
| **Containerization** | Docker | Standard for packaging applications and their dependencies. |
| **Orchestration** | Kubernetes | For automated deployment, scaling, and management of containerized services. |
| **API Gateway** | Traefik / NGINX Ingress | Provides mTLS termination, load balancing, and routing in a Kubernetes environment. |
| **Cloud Provider** | Sovereign Cloud Provider (e.g., Ctrls, E2E Networks) | To ensure data residency and sovereignty as per project requirements. |

---

## 3. Backend Architecture

The system is designed as a set of loosely coupled microservices that communicate via internal APIs. This allows for independent scaling and development of each component.

```ascii
                                     +--------------------------+
                                     |   Security Operator UI   |
                                     +------------+-------------+
                                                  | (Admin API)
                                                  |
+-------------------------------------------------v--------------------------------------------------+
|                                                                                                    |
|  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐  |
|  │   API Gateway    │      │ Attestation Svc  │      │    Intel Svc     │      │  Device Mgmt Svc │  |
|  │ (mTLS Terminus)  │      │ (Verifies Quotes)│      │ (Ingests IOCs)   │      │ (Manages Devices)│  |
|  └────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘  |
|           │                        │                         │                        │           |
|           │                        └─────────────┬───────────┘                        │           |
|           │                                      │                                    │           |
|           └──────────────────────────────────────┼────────────────────────────────────┘           |
|                                                  │ (gRPC / Internal REST)                          |
|                                                  │                                                 |
|  ┌──────────────────┐      ┌─────────────────────┴─────────────────────┐      ┌──────────────────┐  |
|  │ Device Registry  │◄─────┤          Threat Intel Database          │◄─────┤ Golden Firmware  │  |
|  │  (PostgreSQL)    │      │            (TimescaleDB)                │      │   DB (PostgreSQL)  │  |
|  └──────────────────┘      └─────────────────────────────────────────┘      └──────────────────┘  |
|                                                                                                    |
+----------------------------------------------------------------------------------------------------+
       ▲
       │ (TLS 1.3 with Mutual Authentication)
       │
+------┴---------------------------------------------------------------------------------------------+
|                                                                                                    |
|                                     PARASITE-ENABLED DEVICES (Fleet)                               |
|                                                                                                    |
+----------------------------------------------------------------------------------------------------+
```

---

## 4. API Endpoints

The API Gateway exposes a minimal set of endpoints to the devices. All endpoints require a valid and non-revoked client certificate for authentication.

### 4.1 Attestation API
-   `POST /v1/attestation/challenge`
    -   **Request Body:** `{ "device_serial": "string" }`
    -   **Response Body:** `{ "nonce": "base64_string" }`
    -   **Action:** Generates a new 32-byte nonce, stores it temporarily (e.g., in Redis) associated with the device ID, and returns it to the device.

-   `POST /v1/attestation/response`
    -   **Request Body:** `{ "quote": "base64_string", "signature": "base64_string" }`
    -   **Response Body:** `200 OK` (if valid), `400 Bad Request` (if malformed), `403 Forbidden` (if verification fails).
    -   **Action:** Forwards the request to the Attestation Service for full verification.

### 4.2 Intel API
-   `POST /v1/intel/report`
    -   **Request Body:** `{ "report_payload": "base64_string" }` (The encrypted `ThreatReport` from the device)
    -   **Response Body:** `202 Accepted`
    -   **Action:** Accepts the encrypted blob and places it on a message queue for the Intel Service to process asynchronously. This allows the endpoint to respond quickly without waiting for decryption and database insertion.

---

## 5. Database Schema

### 5.1 `devices` Table (PostgreSQL)
Stores the identity and metadata for each provisioned device.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| `id` | `BIGSERIAL` | `PRIMARY KEY` | Unique integer ID. |
| `serial_number` | `TEXT` | `UNIQUE, NOT NULL` | The unique serial number of the device. |
| `public_key` | `TEXT` | `NOT NULL` | The device's public key (PEM format). |
| `certificate` | `TEXT` | `NOT NULL` | The full device identity certificate. |
| `status` | `TEXT` | `NOT NULL` | e.g., 'TRUSTED', 'QUARANTINED', 'REVOKED'. |
| `firmware_version_id` | `BIGINT` | `FOREIGN KEY` | Points to the currently installed firmware version. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | When the device was first registered. |
| `last_seen` | `TIMESTAMPTZ` | | Last successful communication. |

### 5.2 `golden_firmware` Table (PostgreSQL)
Stores the "golden" measurements for each valid firmware version.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| `id` | `BIGSERIAL` | `PRIMARY KEY` | Unique integer ID. |
| `version_string` | `TEXT` | `NOT NULL` | e.g., "v1.2.3". |
| `security_counter` | `INTEGER` | `NOT NULL` | The anti-rollback version number. |
| `bootloader_hash` | `BYTEA` | `NOT NULL` | SHA-256 hash of the bootloader. |
| `parasite_hash` | `BYTEA` | `NOT NULL` | SHA-256 hash of the PARASITE core. |
| `application_hash` | `BYTEA` | `NOT NULL` | SHA-256 hash of the main application. |

### 5.3 `threat_events` Table (TimescaleDB Hypertable)
Stores the time-series data of all reported threats.

| Column Name | Data Type | Constraints | Description |
|-------------|-----------|-------------|-------------|
| `time` | `TIMESTAMPTZ` | `NOT NULL` | Time of the event. The hypertable is partitioned by this column. |
| `device_id` | `BIGINT` | `FOREIGN KEY` | The device that generated the report. |
| `source` | `TEXT` | `NOT NULL` | e.g., 'SENTINEL', 'MEMMANAGE_FAULT'. |
| `address` | `BIGINT` | | The memory address associated with the threat. |
| `program_counter`| `BIGINT` | | The PC at the time of the fault, if available. |
| `metadata` | `JSONB` | | Additional context about the event. |

---

## 6. Service Logic Example (Go)

This conceptual Go snippet shows the core logic inside the **Attestation Service** for handling a response.

```go
// services/attestation/handler.go

package attestation

import (
    "crypto"
    "crypto/ecdsa"
    "crypto/sha256"
    "encoding/base64"
    "encoding/pem"
    "fmt"
    "github.com/gin-gonic/gin"
    "parasite/db" // Assume a db access package
)

// The expected structure of the JSON request body
type AttestationResponse struct {
    Quote     string `json:"quote"`
    Signature string `json:"signature"`
}

// HandleAttestationResponse is the Gin handler for the /response endpoint
func HandleAttestationResponse(c *gin.Context) {
    var req AttestationResponse
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": "Bad request format"})
        return
    }

    // 1. Get device identity from the mTLS client certificate (provided by the gateway)
    deviceSerial := c.GetString("client_serial")
    device, err := db.GetDeviceBySerial(deviceSerial)
    if err != nil {
        c.JSON(403, gin.H{"error": "Device not registered"})
        return
    }

    // 2. Decode the request data
    quoteBytes, _ := base64.StdEncoding.DecodeString(req.Quote)
    signatureBytes, _ := base64.StdEncoding.DecodeString(req.Signature)

    // 3. Hash the quote data - this is what was signed
    quoteHash := sha256.Sum256(quoteBytes)

    // 4. Parse the device's public key
    block, _ := pem.Decode([]byte(device.PublicKey))
    pubKey, _ := x509.ParsePKIXPublicKey(block.Bytes)
    ecdsaPubKey := pubKey.(*ecdsa.PublicKey)

    // 5. Verify the ECDSA signature
    if !ecdsa.VerifyASN1(ecdsaPubKey, quoteHash[:], signatureBytes) {
        c.JSON(403, gin.H{"error": "Invalid signature"})
        // Here, we would also flag the device for suspicious activity
        return
    }

    // 6. Deserialize the quote and perform logical checks
    // (Check nonce, compare hashes against golden_firmware table, etc.)
    // ...
    
    // If all checks pass:
    c.JSON(200, gin.H{"status": "OK"})
}
```
This design provides a scalable, secure, and maintainable backend capable of supporting the PARASITE mission.
