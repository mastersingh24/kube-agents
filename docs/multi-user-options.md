# Architecture & Design Document: Multi-User Hermes Agent Orchestration

This document outlines the architectural patterns for deploying Hermes Agent in environments serving multiple authenticated identities (clients). While this document uses Google Chat as the primary integration example, the architecture is designed to be agnostic to the communication medium.

---

## 1. Core Identity & Authentication Philosophy

To ensure security and proper scoping of MCP (Model Context Protocol) 3-legged OAuth flows, the system must maintain a strict separation between **Platform Identity** (the user in the chat/app) and **Agent Profile** (the Hermes configuration).

- **Identity Mapping:** The system must implement a middleware layer that resolves an incoming `Client_ID` (e.g., Google Chat `users/{id}`) to a specific `Agent_Profile`.
- **Credential Isolation:** Credentials, specifically OAuth tokens for MCP servers, must be stored within a containerized or directory-isolated scope linked to the `Agent_Profile`.

---

## 2. Design Options & Trade-offs

### Approach A: The Resident Profile Pattern (Stateful)

In this model, Hermes Agent instances are pre-provisioned as "resident" profiles. Each user/client is assigned a unique, long-lived profile that persists state, memory, and tokens.

- **Mechanism:** Routing middleware identifies the `Client_ID` and executes Hermes commands with the `HERMES_HOME` environment variable pointed to the specific profile directory.
- **Pros:**
- **Context Retention:** The agent maintains long-term memory and session state.
- **Performance:** No overhead for environment initialization; the agent is "always on."

- **Cons:**
- **Storage Overhead:** Requires managing $N$ profiles on the filesystem.
- **Housekeeping:** Requires processes to manage the lifecycle (creation/deletion) of profiles.

### Approach B: The Ephemeral Execution Pattern (Stateless)

In this model, the agent environment is generated on-demand. A controller initializes a "blank" Hermes environment, injects secure credentials from a central vault, performs the task, and destroys the environment.

- **Mechanism:** Middleware authenticates the `Client_ID`, fetches associated tokens from a secure backend (e.g., Vault), and spins up an isolated `HERMES_HOME` for the duration of the request.
- **Pros:**
- **High Security:** Tokens are never at rest on the disk for long periods.
- **Compliance:** Ideal for environments with strict "zero-persistence" mandates.

- **Cons:**
- **Latency:** Increased cold-start time per request due to setup/teardown.
- **Context Loss:** Minimal to no cross-query memory unless an external vector database is integrated.

---

## 3. Comparison Matrix

| Feature                 | Resident Profile (Stateful)      | Ephemeral Execution (Stateless)    |
| ----------------------- | -------------------------------- | ---------------------------------- |
| **Persistence**         | Native (Full Memory)             | External / None                    |
| **Cold Start**          | Negligible                       | Moderate (System setup)            |
| **Operational Effort**  | Low (Persistent state)           | High (Requires Vault/Orchestrator) |
| **Credential Security** | FS-level isolation (Permissions) | Vault-based (Zero-at-rest)         |

---

## 4. Implementation Strategy

Regardless of the approach, the following components are required for a multi-client deployment:

1. **Identity Middleware:** An authentication service that verifies the user (e.g., via Google Workspace OIDC) before allowing the request to hit the Hermes agent.
2. **Environment Context Injection:** The logic that sets `HERMES_HOME`.

- _Example (Bash-style pseudocode):_

```bash
# Identify user
PROFILE_DIR=$(lookup_profile_path $CLIENT_ID)
# Execute in context
HERMES_HOME=$PROFILE_DIR hermes chat "Analyze my spreadsheet"

```

```
3.  **The "Paste-back" Orchestrator:** When a user initiates a 3-legged OAuth flow, the orchestration layer must catch the manual callback URL from the user and route it to the correct `HERMES_HOME`. This ensures the token is placed in the specific profile folder belonging to that client.

---

## 5. Security & Scaling Recommendations

*   **File System Security:** In Resident configurations, use OS-level `chown` or `chmod` to ensure that even if a process is compromised, the agent cannot read another user's `mcp-tokens/` directory.
*   **Centralized Secret Management:** For high-security environments, move toward a hybrid model where tokens are encrypted at rest using a key per user profile.
*   **Observability:** Implement logging that tracks which `Client_ID` initiated an MCP tool call to maintain a clear audit trail for compliance.

**Follow-up:** Does your current infrastructure already have a centralized identity provider (like Okta, Auth0, or Google Identity) that we should use for the mapping layer, or are you looking to build the mapping logic from scratch?

```
