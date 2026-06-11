# Architectural Analysis: kube-agents vs. GCP Scion

This document analyzes the overlap between the **Kubernetes Agentic Harness (`kube-agents`)** and the **Scion (`googlecloudplatform/scion`)** multi-agent testbed. It evaluates what we gain by adopting Scion versus the friction it introduces, and proposes a path forward.

---

## 1. What is Scion?

**Scion** is an open-source "hypervisor for agents." It is designed to manage the lifecycle of multiple concurrent, isolated deep agents.

- **Isolation:** Runs each agent in a container with strict separation of credentials, configuration, and environment.
- **Workspace Management:** Natively uses `git worktree` to provide each agent with a dedicated workspace, preventing merge conflicts in shared repositories.
- **Harness Bootstrapping:** Uses "Harness Configs" to dynamically mount configurations (like `SOUL.md` and skills) into generic runner images.
- **Multi-Runtime:** Supports Docker, Apple Virtualization, and (Experimental) Kubernetes.

---

## 2. The Overlap: Where We Are Duplicating Effort

If we continue with our custom GKE operator (`platform-agent-operator`) and config-baked Dockerfiles, we are essentially building a custom, GKE-specific version of what Scion does:

- **Config Bootstrapping:** We are struggling with how to get `SOUL.md` and skills into the agent container without baking them. Scion has already solved this via Harness Config mounts.
- **Agent Lifecycle:** Our `platform-agent-operator` manages the creation, deletion, and status of agent Pods. Scion's K8s provider does this via its own orchestrator logic.
- **Workspace Isolation:** Currently, our `devteam` agent clones repositories into a single shared PVC directory. If we run concurrent tasks, they will conflict. Scion's `git worktree` model is the correct solution for this.

---

## 3. What We Gain by Adopting Scion

- **Clean Separation of Code & Config:** We can discard our complex `Dockerfile` and use Scion's generic runner images. Personas (`SOUL.md`) and skills become dynamic configs managed by Scion.
- **Conflict-Free Workspaces:** Natively get `git worktree` isolation for subagents, allowing multiple agents to work on the same repository concurrently without collision.
- **Harness Agnosticism:** We can run the same platform on different underlying agent frameworks (Gemini CLI, Claude Code, etc.) by just changing the Scion profile.
- **Developer Experience:** Natively get `scion attach` (via tmux) to debug agents live in the cluster.
- **Built-in Messaging Integrations:** Scion has a pluggable Message Broker that natively supports **Google Chat** and **Telegram**, meaning we don't need to maintain our own GChat-to-agent gateway code.

---

## 4. Friction & Risks of Adopting Scion

- **Experimental Kubernetes Provider:** Scion's Kubernetes support is experimental. It may not support GKE-specific features natively, such as:
  - GKE Autopilot constraints.
  - Workload Identity (mapping KSA to GSA).
  - Config Connector (KCC) integration for provisioning GCP resources (like Pub/Sub).
- **Push vs. Pull Model:**
  - `kube-agents` is designed as a **declarative (Pull)** GitOps operator. You apply a `PlatformAgent` CRD, and the operator ensures it runs.
  - `Scion` is primarily a **CLI-driven (Push)** tool (e.g., `scion start`). Running Scion _inside_ GKE means the Platform Agent would need to execute the `scion` CLI, requiring K8s API access inside the container and dependency on the Scion binary.
- **Infrastructure Provisioning:** While Scion handles the _connectivity_ to GChat/Telegram via its Message Broker, we still need to provision the underlying GCP resources (Pub/Sub topics, subscriptions, and IAM bindings) that the broker uses. Our custom operator or Terraform is still needed for this infra setup.

---

## 5. Recommendation: The "Scion-Aligned" Path

Instead of a full adoption (which carries risk due to Scion's experimental K8s provider), we should **align our architecture with Scion's patterns** without adopting the Scion orchestrator binary yet.

### Step 1: Adopt the Scion Concept (Generic Image + Mounted Config)

- Refactor our `Dockerfile` to build a single, generic `kube-agent-base` image (installing only `hermes`, `kubectl`, `gcloud`, `gh`, and python dependencies).
- Use Kubernetes **Init Containers** or **ConfigMaps** to mount the `SOUL.md` and skills at runtime, copying Scion's bootstrapping model.

### Step 2: Adopt Git Worktree Isolation

- Modify the `devteam` and `operator` skills to use `git worktree` for repository checkouts instead of raw `git clone` to a single folder, preparing them for Scion-style concurrency.

### Step 3: Keep the Custom Operator (with Scion CRD Integration)

- Retain the `platform-agent-operator` to manage the GKE-specific lifecycle, Workload Identity, and KCC resources. This ensures stability on GKE.
- **Future Integration:** We can modify Scion's Kubernetes provider to deploy our custom `PlatformAgent` resources instead of bare Pods. This allows Scion to serve as the CLI/orchestrator client while our operator handles the complex, secure GKE/GCP provisioning on the backend.

### Benefit of this Path:

This fixes our current architecture issues (no config baking, clean workspaces) immediately using standard Kubernetes patterns, while making the codebase **fully compatible** with Scion. If Scion's Kubernetes provider becomes stable in the future, we can migrate to it with almost zero refactoring of our agent definitions.
