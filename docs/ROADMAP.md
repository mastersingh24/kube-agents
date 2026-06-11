# Roadmap & Cleanup Plan: GKE Multi-Agent Harness

This document outlines the execution plan for cleaning up, restructuring, and testing the `kube-agents` repository. The goal is to align the codebase with production GKE deployments, establish multi-environment testing, improve observability, simplify installation, make agent definitions customizable, and address end-user security and interface diversity.

Each phase is classified as either **[MANDATORY]** (essential for security, stability, or core function) or **[DISCUSSION OPEN]** (concept is needed, but implementation details, names, or tooling are open for debate).

---

## Proposed Folder Structure **[MANDATORY IN PRINCIPLE / NAMIING DISCUSSION OPEN]**

To resolve configuration duplication and clarify boundaries, we propose reorganizing the repository as follows. The _structure_ (separating blueprints from deploy code) is mandatory, but the exact _folder names_ are open for discussion:

```
kube-agents/
├── config/                 # Agent Blueprints (Source of Truth for Personas & Skills)
│   ├── agents/             # Static / Bootstrapped Agent Configurations
│   │   └── platform/       # Platform Agent Configuration
│   │       ├── SOUL.md
│   │       ├── AGENTS.md
│   │       ├── config.yaml
│   │       └── skills/     # Platform-specific skills (e.g., provisioners)
│   │
│   └── templates/          # Base templates used to dynamically provision subagents
│       ├── operator/       # Operator Agent Template
│       │   ├── SOUL.md
│       │   ├── AGENTS.md
│       │   ├── cron/       # Default cron definitions (jobs.json)
│       │   └── skills/     # Operator-specific skills (observability, reliability)
│       └── devteam/        # DevTeam Agent Template
│           ├── SOUL.md
│           ├── AGENTS.md
│           └── skills/     # DevTeam-specific skills (troubleshooting)
│
├── deploy/                 # Deployment Infrastructure (No agent configs baked here)
│   ├── docker/
│   │   └── Dockerfile      # Generic base agent runner Dockerfile (no configs copied)
│   ├── helm/
│   │   └── platform-harness/ # Helm chart for one-click installation
│   └── kustomize/          # Raw K8s manifests (alternative to Helm)
│
├── integrations/           # Third-party connectivity setups
│   └── gchat/              # GChat integration resources (operator, KCC Pub/Sub manifests)
│
├── local-dev/              # Local offline development & Kind testing
│   ├── kind-config.yaml    # Kind cluster setup
│   └── bootstrap.sh        # Bootstraps local registry, Kind, and Gitea
│
├── docs/                   # Documentation & Guides
│   ├── ROADMAP.md          # Project roadmap (this file)
│   ├── m1-demos.md         # Walkthrough scenarios
│   └── architecture.md
│
├── README.md
├── INSTALL.md
└── AGENTS.md               # Repository contribution guidelines
```

---

## Phase 1: Repository Cleanup & Alignment

### 1.1. Archive & Prune **[MANDATORY]**

- **Action:** Create a Git branch `archive/v1-local-harness` to preserve the historical local-execution files.
- **Action:** Remove the obsolete `workspace/` files that target local execution runners (like simple local shell scripts or local-only heartbeat configurations).
- **Action:** Keep the dynamic provisioning templates, now relocated to `config/templates/`, as they are required by the Platform Agent for dynamic provisioning.

### 1.2. De-duplicate Configurations (Generic Image Model) **[MANDATORY]**

- **Action:** Refactor `agents/Dockerfile` to build a single `kube-agent-base` image containing only the runtime (`hermes-agent`), CLI tools (`kubectl`, `gcloud`, `gh`), and Python/Node.js dependencies.
- **Action:** Externalize agent configurations. Instead of baking `SOUL.md` and `skills/` into the Docker image, modify the deployment manifests to mount these files at runtime:
  - Use Kubernetes **ConfigMaps** to hold the agent-specific `SOUL.md`, `AGENTS.md`, and `cron/jobs.json`.
  - Use Kubernetes **Init Containers** to clone the target configuration/skills repository into a shared PVC volume on boot.
- _(Note: See [scion-duplication.md](scion-duplication.md) for the architectural analysis on why we are moving away from config-baked images)._

---

## Phase 2: Multi-Environment Deployment & Tenant Isolation

To support continuous development without disrupting users, we will establish two isolated environments.

```
                   [Feature Branches]
                           │
                           ▼ (Pull Requests)
                     [dev Branch] ───► Deploy to Continuous Testing Env (GKE/Kind)
                           │
                           ▼ (Release Merge)
                    [main Branch] ───► Deploy to Stable / Trial User Env (GKE)
```

### 2.1. Git Branching Strategy **[MANDATORY]**

- **`dev` Branch:** The integration branch for continuous testing. All feature PRs target `dev`. Automated testing (Kind) runs against this branch.
- **`main` Branch:** Reflects the known good, stable state. Code is promoted from `dev` to `main` only after validation.

### 2.2. Stable / Trial Environment Isolation **[MANDATORY]**

To onboard new users without "crossing wires" or creating noisy chat spaces:

- **Workspace Isolation:** Assign each trial group their own **Kubernetes Namespace** (e.g., `agent-system-team-a`).
- **GitOps Repository Isolation:** Provide each group with a dedicated Git repository (or an isolated directory in a monorepo) for their application manifests.
- **Messaging Space Isolation (Shared Ingress Topic + Topic Splitter):**
  - Because we are constrained to a **single Pub/Sub topic** for GChat ingress, and standard Google Chat integrations do not support custom Pub/Sub message attributes for subscription-level filtering, we cannot subscribe Platform Agents directly to the main ingress topic without message collision.
  - **The Solution [DISCUSSION OPEN]:** Implement a **Central Topic Splitter (Chat Router)**. This component (e.g., a lightweight Cloud Run service or in-cluster deployment) consumes from the main GChat ingress topic, decodes the payload to extract the space ID or sender email, and **re-publishes** the message to a **dedicated, tenant-specific Pub/Sub topic**.
  - **Agent Pull Model Consistency:** Each Platform Agent maintains its standard configuration, subscribing directly to its dedicated tenant topic. This preserves the pull-based subscriber model and keeps the agent configuration identical to a single-tenant installation.
  - **Private DMs & Space Support:** The Splitter maps GChat Space IDs and user emails (for private DMs) to their respective tenant Pub/Sub topics, ensuring clean isolation without crossing wires.

---

## Phase 3: Documentation & Walkthrough Scenarios

- **Action:** Rewrite the root `README.md` and `INSTALL.md` to document the new containerized, ConfigMap-mounted architecture, removing all references to the obsolete local harness. **[MANDATORY]**
- **Action (Static Docs Site):** Scaffold a basic **Hugo-based** (or MkDocs/Docusaurus) documentation site under `docs/` to host architecture diagrams, user guides, and SRE playbooks. **[DISCUSSION OPEN]**
  - _Justification:_
    - **Docs-as-Code:** Allows developers to write documentation in Markdown alongside the code. The site generator renders them directly, eliminating duplicate maintenance effort.
    - **Branch-Aligned Versioning:** The docs site is versioned with the branch (e.g., the `dev` branch documentation site automatically reflects experimental features, while `main` reflects only stable releases).
    - **Low Maintenance:** Hosting a static site (via GitHub Pages or a GKE Nginx Pod) is cost-efficient and requires zero database administration or security patching.
    - **Continuous Updates:** Encourages documentation updates as a mandatory step in the PR review hygiene process.
- **Action:** Create a `docs/demos/` directory containing step-by-step walkthrough guides based on the 5 validated scenarios. We have bootstrapped these scenarios in [m1-demos.md](m1-demos.md) in the repository root. **[DISCUSSION OPEN]** _(The specific walkthrough scenarios and formatting can be tailored based on team feedback)._
  1.  _GKE Cluster Drift Detection & GitOps PR Creation_
  2.  _Automated Workload-Aware CVE Scan & Escalation_
  3.  _DevTeam Workload Diagnostics & Fix (OOMKilled)_
  4.  _Operator Health Patrol & Scheduling Failure Alerting_
  5.  _Dynamic DevTeam Agent Onboarding_

---

## Phase 4: Observability & Telemetry (Audit Trail)

We must record all interactions to learn what users are requesting and how agents are behaving.

- **Action (Structured Logging):** Configure the agent runner (`hermes`) to output structured JSON logs for key SRE events (prompts, tool calls, PR creations). **[MANDATORY]**
- **Action (Centralized Auditing):** Deploy a log aggregation agent (e.g., Cloud Logging agent in GKE, Fluentbit in Kind) to ship these logs to a central console. **[MANDATORY]**
- **Action (Memory Archive):** Set up a job to periodically backup the PVC-backed `memory/` directories containing the agent's long-term context (`MEMORY.md` and `heartbeat-state.json`) for analysis. **[DISCUSSION OPEN]** _(Exact backup destination and schedule can be defined later)._

---

## Phase 5: Self-Install & One-Click Deployment (Kind & Real Cluster)

We need a simple path for users to install the harness on their own clusters.

### 5.1. Operator vs. Helm Decision **[DISCUSSION OPEN]**

- **The Recommendation:** Use a **hybrid model**:
  - **Helm/Kustomize:** Used for the _initial static installation_ (installing the Platform Agent, CRDs, RBAC, Secrets, and the GChat/Telegram Message Broker).
  - **Operator:** Kept for the _dynamic runtime lifecycle_ (the Platform Agent dynamically creating `devteam` and `operator` custom resources, which are then reconciled into Pods).
- _Alternative:_ Explore a pure Helm-based installation or a pure Custom Operator installation.

### 5.2. Local Testing with Kind **[MANDATORY]**

- **Action:** Create a `local-dev/` directory containing scripts to spin up a local [Kind](https://kind.sigs.k8s.io/) cluster, install a local registry, deploy a mock Git server (e.g., Gitea), and install the harness for offline testing.

---

## Phase 6: Flexible & Customizable Agent Definitions

Agent definitions are currently too rigid. Users should be able to opt out of our defaults and inject their own best practices.

- **Action (Modular Rules):** Split the agent configuration so that the core persona (`SOUL.md`) is immutable, while user-specific rules are loaded from a separate ConfigMap and merged on boot. **[MANDATORY]**
- **Action (Opt-in/Opt-out Skills):** Add feature flags in `config.yaml` to toggle default cron tasks (e.g. `cve-scan`). **[DISCUSSION OPEN]** _(Can start with all defaults enabled and add toggles incrementally)._

---

## Phase 7: End-User Granular Security & Identity

When operating in shared chat spaces or multi-user clients, we must prevent "Confined Escalation" where unauthorized users trigger actions using the agent's elevated cluster credentials.

### 7.1. Identity Propagation **[MANDATORY]**

- **Action:** Configure the messaging connector (GChat/Telegram Broker) to propagate the sender's identity (email or ID) in the request payload metadata.

### 7.2. Authorization Enforcement Options **[DISCUSSION OPEN]**

To map incoming user identities to cluster permissions, we will investigate two approaches:

- **Option A: Token-per-Role/User Mapping (Simple/Initial)**
  - Map distinct API tokens to specific user emails/roles in the Platform config. The client assigns the corresponding token to the user, and the agent authorizes them based on the token.
- **Option B: OIDC Proxy Injection (Enterprise Standard)**
  - Deploy an **OAuth2 Proxy** in front of the Platform Agent API. The proxy authenticates users against the corporate IDP, extracts their email from the OIDC token, and injects it as an HTTP header (e.g., `X-User-Email`). The agent reads this header to authorize actions.

### 7.3. Multi-User Environment Scoping **[DISCUSSION OPEN]**

- **Action:** Evaluate how the agent runner configures and executes commands for multiple users, specifically concerning session memory and MCP credential isolation.
- **Reference:** See the detailed options and trade-offs in [multi-user-options.md](multi-user-options.md).
  - **Resident Profile Pattern (Stateful):** Maps users to long-lived `HERMES_HOME` directories to preserve context/OAuth tokens.
  - **Ephemeral Execution Pattern (Stateless):** Dynamically spins up clean environments on-demand, injecting credentials from a vault and destroying the environment after execution.

---

## Phase 8: Alternative Interfaces & UX **[DISCUSSION OPEN]**

Relying solely on public chat spaces can be noisy and exposes security risks. We will explore and enable alternative interfaces supported by the Hermes runner:

### 8.1. OpenAI-Compatible Web Clients

Because Hermes Agent exposes an OpenAI-compliant API server (`/v1/chat/completions`), we can connect standard open-source frontends like **Open WebUI** or **LibreChat** (both support user authentication and custom OpenAI backends).

### 8.2. Official Hermes Web Dashboard (Live PTY)

- **Action:** Expose the built-in Hermes dashboard (`9119`) behind Google **Identity-Aware Proxy (IAP)**.
- **Capabilities:** The dashboard Chat tab embeds a live Terminal User Interface (TUI) via `xterm.js` over WebSockets, allowing real-time interaction and live viewing of the agent's thought process.
- _Multi-Pod Architecture Limitation:_ Because agents run in separate Pods, the dashboards are isolated. To view the Operator Agent's live PTY, the user must connect to the Operator Pod's dashboard port.
