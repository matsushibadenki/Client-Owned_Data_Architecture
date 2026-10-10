# Client-Owned Data Architecture

**Services provide the applications. Users own their data and authentication credentials.**

Client-Owned Data Architecture (CODA) is a proposed shared authentication and data-access foundation for web services. A local encrypted vault holds the authoritative copy of each user's data. Services receive access only during an authenticated, explicitly authorized session.

**Project status:** Design stage. This repository currently contains specifications and an architecture diagram; the application, browser extension, and SDK are not implemented yet.

## Architecture

![CODA architecture: service applications connect through a browser extension and Native Messaging to a local Rust vault. Only an active session and a service-signed operation can change the user's encrypted data.](docs/assets/coda-architecture.png)

The diagram shows the proposed desktop architecture. The browser extension establishes the local bridge and verifies the actual page origin. The Vault Core opens an outbound TLS connection to the registered service API, authenticates the service, and enforces permissions on every operation. Services never receive database files or vault encryption keys.

## Core principles

- **Client-owned storage:** Profiles, documents, history, and settings remain in service-specific encrypted databases on the user's device.
- **Shared foundation:** Independent services can integrate with the same vault protocol while keeping their data namespaces and encryption keys separate.
- **Connection-required changes:** Creating, updating, deleting, or importing application data requires an active connection to the relevant service and a service-signed operation.
- **No direct local editing:** Unlocking the vault does not grant editing rights. The vault management UI, local tools, and browser extension cannot independently change application data.
- **Session-limited access:** Logout, revocation, locking, detected disconnection, and expiration stop further authorized access.
- **Continuous local persistence:** Each accepted change is committed locally before it is reported as saved. Persistence does not depend on logout.
- **Minimal disclosure:** Services receive only the fields and operations covered by the user's consent.

## How a session works

1. The user opens a service and authenticates with a passkey.
2. The browser extension verifies the page origin and forwards a short-lived connection ticket to the vault.
3. The Vault Core verifies the service, unlocks through OS authentication, and requests consent when needed.
4. The service API validates each change and signs the operation. Its signing private key stays on the server.
5. The Vault Core verifies the active connection, session, permissions, signatures, and expected data revision before committing the change.
6. The service receives a commit receipt. After disconnection or expiration, further changes require a new authorized session.

Connecting to Service A does not grant access to Service B's data. Offline editing and deferred offline writes are outside the MVP.

## Data integrity and recovery

The design combines encrypted storage with persistent service signatures and a signed manifest of each service's records. Both the Vault Core and the service API verify data integrity, so locally modified content cannot be accepted simply because a modified client submits it.

Encrypted backups can be created and inspected locally. Restored data remains quarantined until the relevant service reconnects, verifies it, and signs a recovery approval. Restoration is not an alternative path for offline editing.

Key protection, backup recovery, and signed data verification have separate roles. A passkey is not used directly as a database encryption key.

## Proposed components

| Component | Responsibility |
| --- | --- |
| Service web applications and APIs | Business UI, user authentication, input validation, signed changes, transient processing |
| Browser extension and Native Messaging host | Verified origin context and a restricted bridge to the local core |
| Rust Vault Core | Authoritative state, authorization, sessions, keys, integrity verification, persistence, and audit records |
| Tauri 2 management UI | Consent, connection management, storage status, backup, and recovery; dockable and floating panels |
| Service-specific SQLCipher databases | Encrypted local application data with independent keys |
| Service registry | Trusted service identifiers, allowed origins, public keys, and revocation information |

All management windows use state owned by Rust. If intensive image processing or GPU rendering is introduced, it will run in Rust with wgpu; large frame buffers remain on the Rust/GPU side.

The proposed initial validation target is macOS with a Chromium-based browser. Windows, Linux, Safari, mobile, portable SSD workflows, and multi-device synchronization are planned extensions. Product UI localization targets English, Japanese, and Simplified Chinese.

## Security boundaries

CODA aims to prevent unauthorized vault access and reject unsigned or altered application data. It cannot physically prevent a device owner from modifying or deleting files, or guarantee protection after complete compromise of the operating system.

Once plaintext is delivered to a service, cryptography cannot guarantee that the service has not copied it. Service-side non-retention therefore also requires implementation controls and operational verification. Sensitive data should be processed locally where practical.

Valid signatures do not prove that a snapshot is the newest one. An independent freshness checkpoint is required to reliably detect a complete rollback to an older, correctly signed snapshot. The checkpoint design remains an implementation prerequisite.

The proposed service policy avoids persistent application data. Whether to retain minimal authentication metadata and freshness checkpoints is an open design decision. Public posts, payments, inventory, and other shared transactional state are outside the MVP.

## Documentation

The detailed specifications are currently maintained in Japanese.

| Document | Contents |
| --- | --- |
| [Design specification](docs/design-specification.md) | Requirements, trust boundaries, authentication, authorization, storage, recovery, interfaces, and acceptance criteria |
| [Roadmap](docs/roadmap.md) | Implementation phases, completion conditions, and validation dependencies |
| [Documentation index](docs/README.md) | Documentation scope and source context |

## Roadmap

The labels below describe planned work. No application features are currently marked as implemented.

| Status | Work |
| --- | --- |
| 🟠 [Next] | Finalize protocol contracts, signature formats, authentication metadata policy, and freshness checkpoints |
| 🟠 [Next] | Implement the Rust core, encrypted service databases, and connection-required signed mutations |
| 🔴 [Later] | Add session authorization, browser integration, and two independent demo services |
| 🔴 [Later] | Build the Tauri management UI, localization, encrypted backups, and approved recovery |
| ⭕️ [Pending] | Validate hardware key protection, power-loss recovery, production non-retention, infrastructure, and an independent security audit |
| 🔴 [Later] | Expand platform support, portability, synchronization, consent-based sharing, and interoperability |

🟢 [Done] means implemented in the current codebase. 🟠 [Next] means high-priority unfinished work. 🔴 [Later] means planned work beyond the immediate next step. ⭕️ [Pending] means validation is on hold until the required environment is available.

## Repository contents

```text
.
├── .gitignore
├── README.md
└── docs/
    ├── README.md
    ├── design-specification.md
    ├── roadmap.md
    └── assets/
        └── coda-architecture.png
```

There are no installation or build commands yet. Follow the design specification and roadmap when implementation begins. Local vaults, runtime databases, backups, environment files, private keys, and build artifacts are excluded by `.gitignore`.
