# M1 Demonstration Scenarios

This document outlines 5 realistic demonstration scenarios to showcase the capabilities of the Platform, Operator, and DevTeam agents in the Kubernetes Agentic Harness. These scenarios are designed based on the actual skills, scripts, and SOPs present in the repository.

---

## Demo 1: Platform Agent - GKE Cluster Drift Detection & GitOps Reconciliation (Blueprint Sync)

- **Target Agent:** Platform Agent (`platform`)
- **Purpose:** Showcase fleet-wide governance, drift detection, and GitOps-enforced self-healing.
- **Setup Steps (Simulating Drift):**
  1.  Target a managed GKE cluster (e.g., `mercury-01`).
  2.  Manually modify the live cluster configuration to violate the platform blueprint. For example, remove the mandatory annotation that removes the default node pool:
      ```bash
      kubectl annotate containercluster mercury-01 cnrm.cloud.google.com/remove-default-node-pool- --overwrite
      ```
  3.  _(Alternative)_: Manually disable Autopilot on the live cluster if using GKE Config Connector.
- **Execution (Trigger):**
  Ask the Platform Agent in chat:
  > "Run the daily Blueprint Sync audit."
- **Agent Workflow:**
  1.  The Platform Agent queries the GKE Operator for `mercury-01` to retrieve the live cluster manifest.
  2.  It detects that the `remove-default-node-pool` annotation is missing, violating the blueprint.
  3.  Following the GitOps boundary (no direct mutations), it uses the `submit-suggestion` skill to clone the infrastructure repository, restore the annotation in the YAML file, push a branch, and open a GitHub Pull Request (PR).
- **Expected Proof:**
  The Platform Agent returns the **GitHub Pull Request URL** showing the proposed fix.

---

## Demo 2: Operator Agent - Automated CVE Scan and Escalation

- **Target Agent:** Operator Agent (`operator`)
- **Purpose:** Showcase background vulnerability scanning and structured SRE escalation.
- **Setup Steps:**
  1.  Deploy an old, vulnerable container image to the cluster:
      ```bash
      kubectl run vulnerable-app --image=nginx:1.17.6 --namespace=default
      ```
- **Execution (Trigger):**
  Ask the Operator Agent in chat:
  > "Execute the scheduled CVE Scan."
- **Agent Workflow:**
  1.  The Operator lists all running images in the cluster and identifies `nginx:1.17.6`.
  2.  It queries GKE Security Posture or Artifact Registry for vulnerabilities associated with this image.
  3.  It detects `CRITICAL` or `HIGH` severity CVEs.
  4.  It executes the Escalation Protocol: runs a quick 3-step Root Cause Analysis (RCA), assesses the Security impact, and formats a structured report.
  5.  It sends this report directly to the `@platform` agent.
- **Expected Proof:**
  A message in the Platform Agent's logs/inbox containing the structured CVE report, identifying `vulnerable-app` in namespace `default`, and suggesting the exact `kubectl set image` command (or GitOps change) to update it.

---

## Demo 3: DevTeam Agent - Automated Workload Troubleshooting and GitOps Fix (OOMKilled)

- **Target Agent:** DevTeam Agent (`devteam`)
- **Purpose:** Showcase logs/events diagnostics, root-cause analysis, and autonomous GitOps PR creation.
- **Setup Steps:**
  1.  Deploy a workload designed to crash due to Out-Of-Memory (OOM) with limits set too low:
      ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: memory-hog
        namespace: payments
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: memory-hog
        template:
          metadata:
            labels:
              app: memory-hog
          spec:
            containers:
              - name: main
                image: busybox
                command: ["sh", "-c", "x=a; while true; do x=$x$x; done"]
                resources:
                  limits:
                    memory: "10Mi"
      ```
  2.  Wait for the pod to enter `CrashLoopBackOff` due to `OOMKilled` (Exit Code 137).
- **Execution (Trigger):**
  Ask the DevTeam Agent in chat:
  > "@devteam-payments Troubleshoot the memory-hog deployment in namespace payments."
- **Agent Workflow:**
  1.  The DevTeam agent queries the pod status and sees `CrashLoopBackOff`.
  2.  It inspects the container's `lastState.terminated` status and finds `ExitCode: 137`.
  3.  It identifies this as an OOM event.
  4.  It clones the application repository, creates a branch, increases the memory limit in the deployment manifest (e.g., from `10Mi` to `128Mi`), commits the change, and pushes a PR.
- **Expected Proof:**
  A chat message explaining the RCA (Exit Code 137/OOM) and providing the **GitHub PR URL** containing the manifest fix.

---

## Demo 4: Operator Agent - Health Patrol & Alerts (Scheduling Failures)

- **Target Agent:** Operator Agent (`operator`)
- **Purpose:** Showcase cluster capacity diagnostics and proactive escalation of unschedulable workloads.
- **Setup Steps:**
  1.  Deploy a Pod that requests more CPU than any single node in your cluster can provide, forcing it to remain `Pending`:
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: massive-pod
        namespace: payments
      spec:
        containers:
          - name: main
            image: nginx
            resources:
              requests:
                cpu: "100"
      ```
- **Execution (Trigger):**
  Ask the Operator Agent in chat:
  > "Perform a cluster health check."
- **Agent Workflow:**
  1.  The Operator scans the cluster and finds `massive-pod` in `Pending` state.
  2.  It queries events and identifies the `FailedScheduling` signature: `Insufficient cpu`.
  3.  Since it cannot auto-remediate this (as it violates capacity limits), it escalates the issue to the Platform Agent with an assessment of the Reliability risk (workload cannot start).
- **Expected Proof:**
  An escalation report sent to the `@platform` agent detailing the resource pressure and identifying the specific pod and namespace.

---

## Demo 5: Platform Agent - Dynamic DevTeam Provisioning

- **Target Agent:** Platform Agent (`platform`)
- **Purpose:** Showcase automated tenant onboarding and dynamic subagent deployment.
- **Setup Steps:**
  Ensure you have an application repository available for onboarding (e.g., `git@github.com:your-org/analytics-app.git`).
- **Execution (Trigger):**
  Ask the Platform Agent in chat:
  > "Provision a new DevTeam agent for namespace 'analytics' with repository 'git@github.com:your-org/analytics-app.git'."
- **Agent Workflow:**
  1.  The Platform Agent validates that all parameters are present.
  2.  It generates a secure Bearer token for inter-agent communication.
  3.  It reads the DevTeam deployment template.
  4.  It replaces placeholders (`<NAMESPACE>`, `<GIT_REPO>`) with the provided parameters.
  5.  It clones the target infrastructure repository, saves the deployment manifest to `k8s/devteam-agent.yaml`, commits the changes, and opens a GitHub PR.
- **Expected Proof:**
  A **GitHub PR URL** containing the deployment manifest for the new `devteam-analytics` agent pod.
