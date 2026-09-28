# 1. "A pod keeps crashing/restarting (CrashLoopBackOff). How do you debug it?"

When a Kubernetes Pod is stuck in CrashLoopBackOff, it means the container starts, crashes, fails, and Kubernetes keeps trying to restart it over and over.

Here is the step-by-step process to debug and fix it:

## STEP 1: CHECK THE POD STATUS

First, check basic details about the Pod, such as its exact name, restart count, and status.

### Command:

    kubectl get pods -n <namespace>

### What to look for:

- Check the RESTARTS column (high number indicates rapid crashing).
- Confirm the exact status is CrashLoopBackOff.

## STEP 2: CHECK CONTAINER LOGS (MOST IMPORTANT STEP)

In most cases, the application crashes because of code errors, missing environment variables, or failed database connections.

### Commands:

    # View current logs
    kubectl logs <pod-name> -n <namespace>

    # If the container already crashed, view logs from the PREVIOUS container instance
    kubectl logs <pod-name> --previous -n <namespace>

    # If the Pod has multiple containers, specify the container name
    kubectl logs <pod-name> -c <container-name> --previous -n <namespace>

### What to look for:

- Error stack traces (e.g., Unhandled Exception, Connection Refused).
- Missing environment variable error messages.

## STEP 3: INSPECT POD EVENTS & EXIT CODES

If logs are empty or the container fails before running code, check Kubernetes system events and exit codes.

### Command:

    kubectl describe pod <pod-name> -n <namespace>

### What to look for:

- Look at the "Containers" section for "Last State" -> "Exit Code".
- Look at the "Events" section at the very bottom.

### Common Exit Codes & Meanings:

- Exit Code 1 / 255 : General application error (check code/config).
- Exit Code 137 : OOMKilled (Out of Memory — container exceeded its memory limit).
- Exit Code 126/127 : Command/entrypoint script not found or not executable.
- Exit Code 139 : Segmentation fault (memory access error).

## STEP 4: VERIFY CONFIGURATION, SECRETS & VOLUMES

If the application relies on external resources, verify that they actually exist in the same namespace.

### Commands:

    # Check referenced ConfigMaps
    kubectl get configmaps -n <namespace>

    # Check referenced Secrets
    kubectl get secrets -n <namespace>

    # Check volume status
    kubectl get pvc -n <namespace>

### What to look for:

- Typos in environment variable names inside the YAML manifest.
- Missing ConfigMaps or Secrets referenced by the Pod.
- Persistent Volumes failing to attach or mount.

## STEP 5: TEST NETWORK & SERVICE CONNECTIVITY INTERACTIVELY

If the app crashes due to network timeouts (e.g., cannot reach a database or another microservice), spin up a temporary container inside the cluster to test DNS and ports.

### Command:

    kubectl run net-test --rm -it --image=busybox -- sh

Inside the busybox shell:

    # Test DNS resolution
    nslookup <service-name>

    # Test database/port connection
    nc -zv <service-name> <port>

# What's the difference between liveness and readiness probes, and how can a misconfigured one cause CrashLoopBackOff?

Liveness restarts the container; readiness removes it from service endpoints. A too-aggressive liveness probe (short timeout/low failure threshold) can kill a slow-starting app repeatedly.

## LIVENESS PROBE VS. READINESS PROBE

| FEATURE | LIVENESS PROBE | READINESS PROBE |
|---|---|---|
| Primary Purpose | Detects if the container is ALIVE or deadlocked/unresponsive. | Detects if the container is READY to accept incoming network traffic |
| Action on Failure | Kills and RESTARTS the container. | Removes Pod IP from Service endpoint (stops traffic, app keeps running) |
| Ideal Use Case | App is stuck in a deadlock and cannot recover without a restart. | App is booting up, warming cache, or temporarily overloaded. |

## HOW A MISCONFIGURED PROBE CAUSES CrashLoopBackOff

When a Liveness Probe is too aggressive, it triggers a continuous restart loop:

1. **SLOW STARTUP:** Your application takes 30 seconds to boot (loading cache, connecting to databases, initializing runtime).

2. **AGGRESSIVE CHECK:** The liveness probe starts checking after 5 seconds (`initialDelaySeconds: 5`) with a low tolerance (`failureThreshold: 2`).

3. **PREMATURE KILL:** The probe fails because the app isn't ready yet. Kubernetes assumes the container is deadlocked and KILLS IT.

4. **THE LOOP:** Kubernetes restarts the container -> App attempts to boot -> Probe fails again -> Container killed again.

Repeatedly failing and restarting causes Kubernetes to flag the Pod with CrashLoopBackOff.

## HOW TO FIX & PREVENT THIS ISSUE

### 1. Adjust Probe Tolerances:

Give slow-starting apps enough buffer using `initialDelaySeconds` and `failureThreshold`.

**Example Manifest Snippet:**

    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30  # Wait 30s before the first health check
      periodSeconds: 10        # Check every 10 seconds
      timeoutSeconds: 5         # Wait 5s for a response before timing out
      failureThreshold: 3       # Require 3 consecutive failures before restarting

### 2. Use a Startup Probe (Best Practice):

For apps that take a long time to start, use a `startupProbe` alongside `livenessProbe`. Kubernetes disables liveness and readiness checks until the startup probe succeeds, preventing premature container restarts.

# "How do resource requests/limits relate to OOMKilled?" → Limits cap memory; exceeding it triggers OOMKill regardless of node capacity.

## RESOURCE REQUESTS VS. LIMITS & OOMKilled

### 1. CORE DEFINITIONS:

- **Requests** : The MINIMUM guaranteed memory/CPU Kubernetes reserves for a pod on a node.
  - Used by the Scheduler to decide which node has room to place the Pod.
- **Limits** : The HARD CAP maximum amount of memory/CPU a container is allowed to consume.

### 2. HOW OOMKilled OCCURS:

- Memory is a non-compressible resource. Unlike CPU (which gets throttled when limits are exceeded), exceeding memory limits causes immediate termination.
- When a container tries to allocate more RAM than specified in 'limits.memory', the Linux Kernel's Out-Of-Memory (OOM) Killer intervenes and instantly terminates the process.
- The Pod exits with Exit Code 137 (OOMKilled) and enters a CrashLoopBackOff state.

### 3. KEY INSIGHT:

- OOMKill occurs strictly based on the container's defined 'limits.memory', NOT on available node capacity.
- Even if the physical Kubernetes worker node has 128 GB of free, unused RAM, a container with a limit of 512Mi will be OOMKilled the second it requests 513Mi.

## EXAMPLE MANIFEST & BEHAVIOR

    resources:
      requests:
        memory: "256Mi"   # Guaranteed 256MB on scheduling
      limits:
        memory: "512Mi"   # Hard ceiling at 512MB

Behavior Scenarios:
- Usage < 256Mi  : Normal operation.
- Usage = 400Mi  : Allowed (between request and limit).
- Usage > 512Mi  : OOMKilled (Exit Code 137) triggered immediately by Linux Kernel.

## HOW TO FIX OOMKilled ISSUES

### 1. Check if Pod was OOMKilled:

    kubectl describe pod <pod-name> -n <namespace>

(Look for: "Last State: Terminated", "Reason: OOMKilled", "Exit Code: 137")

### 2. Increase Memory Limits:

Raise 'limits.memory' in your Deployment/StatefulSet YAML manifest.

### 3. Profile Application Memory:

Investigate memory leaks, unoptimized database queries, or bloated garbage collection settings within application code.

# "How would you check node-level issues vs pod-level issues?" → kubectl describe node, check for pressure conditions (MemoryPressure, DiskPressure).

## POD-LEVEL ISSUES VS. NODE-LEVEL ISSUES

### 1. CORE DIFFERENCE:

- **Pod-Level Issues** : Isolated to a specific container, application code, missing environment variables, or manifest misconfigurations.
- **Node-Level Issues**: Affect all Pods running on that worker node due to resource exhaustion, kubelet failures, or infrastructure faults.

## HOW TO DIAGNOSE NODE-LEVEL ISSUES

### Step 1: Check Node Status and Readiness

**Command:**

    kubectl get nodes

**What to look for:**

- Status column: Look for 'NotReady' or 'SchedulingDisabled'.

### Step 2: Inspect Node Pressure Conditions & System Events

**Command:**

    kubectl describe node <node-name>

**What to look for in "Conditions":**

- MemoryPressure = True : Node is dangerously low on RAM.
- DiskPressure = True : Node root filesystem or image disk is full.
- PIDPressure = True : Too many running processes on the node.
- Ready = False : Kubelet isn't reporting health or network is down.

**What to look for in "System Info & Events":**

- OutOfDisk / OutOfMemory events logged at node scope.
- Kubelet / Container Runtime (Docker/containerd) errors.

## HOW TO DIAGNOSE POD-LEVEL ISSUES

### Step 1: Check Pod Status across the Namespace

**Command:**

    kubectl get pods -n <namespace> -o wide

**What to look for:**

- Status: Pending, CrashLoopBackOff, ImagePullBackOff, Error.
- Node Column: Check if all failing Pods belong to the SAME node (points to Node issue) or are spread across DIFFERENT nodes (points to Pod issue).

### Step 2: Inspect Pod Logs & Configuration

**Commands:**

    kubectl logs <pod-name> -n <namespace> --previous
    kubectl describe pod <pod-name> -n <namespace>

**What to look for:**

- Application stack traces, bad config values, unhandled runtime crashes.
- OOMKilled (Exit Code 137), FailedMount (missing Secret/ConfigMap).

# "How do you debug if kubectl logs shows nothing at all?" → Container may be failing before app starts logging; check init containers, kubectl get events, or exec into a debug container.

## DEBUGGING WHEN `kubectl logs` SHOWS NOTHING AT ALL

When `kubectl logs <pod-name>` returns completely blank output, it means the main application container exited before write streams (stdout/stderr) could capture logs, or the execution never reached your application runtime.

## STEP-BY-STEP DIAGNOSTIC WORKFLOW

### 1. CHECK INIT CONTAINERS FIRST

If an Init Container fails, the main application container will NEVER start.

**Commands:**

    # List all containers in the Pod (including Init Containers)
    kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.initContainers[*].name}'

    # Check logs for a specific Init Container
    kubectl logs <pod-name> -c <init-container-name> -n <namespace>

### 2. INSPECT KUBERNETES SYSTEM EVENTS & POD DETAILS

Cluster-level events report container runtime failures that happen before the app boots.

**Command:**

    kubectl describe pod <pod-name> -n <namespace>

**What to look for:**

- State / Last State: Check Exit Codes (e.g., Exit Code 127 = binary not found).
- Events section (at bottom): Look for:
  - FailedToStartContainer / InvalidImageName
  - PermissionDenied (lacking execution permissions on entrypoint script)
  - FailedMount (missing Secret, ConfigMap, or PersistentVolume)

### 3. CHECK RECENT KUBERNETES EVENTS IN THE NAMESPACE

Get a chronological view of events across the namespace.

**Command:**

    kubectl get events -n <namespace> --sort-by='.metadata.creationTimestamp'

### 4. CHECK FOR ENTRYPOINT / CMD ISSUES IN DOCKERFILE

Common causes of immediate silent crashes:

- Shebang missing or invalid in startup script (e.g., `#!/bin/bash` missing in image).
- Entrypoint binary path is incorrect or missing execution permissions (`chmod +x`).
- Logging is directed to a file inside the container instead of stdout/stderr.

### 5. ATTACH AN INTERACTIVE DEBUG CONTAINER

If the container keeps crashing immediately, run an ephemeral debug container with an interactive shell to inspect the file system, entrypoint scripts, and permissions.

**Command:**

    kubectl debug pod/<pod-name> -it --image=busybox -- target-binary-name


# 2. "How do you deploy a new version with zero downtime, and how do you roll back if it fails?"

## ZERO-DOWNTIME DEPLOYMENT & ROLLBACK IN KUBERNETES

### 1. DEPLOYING A NEW VERSION WITH ZERO DOWNTIME (ROLLING UPDATE)

Kubernetes performs zero-downtime updates by default using the RollingUpdate strategy. It replaces old Pods with new Pods incrementally, ensuring traffic is always served by active Pods.

**Step 1: Configure Deployment Strategy & Health Probes in YAML**

In your Deployment manifest, define the RollingUpdate strategy alongside Readiness Probes. The Readiness Probe guarantees traffic is routed ONLY to new Pods that are fully ready.

**Example Deployment Manifest Snippet:**

    spec:
      replicas: 4
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxSurge: 25%        # Max extra Pods created above target replica count
          maxUnavailable: 25%  # Max Pods that can be offline during update
      template:
        spec:
          containers:
          - name: my-app
            image: myregistry/myapp:v2.0.0
            readinessProbe:
              httpGet:
                path: /healthz
                port: 8080
              initialDelaySeconds: 10
              periodSeconds: 5

**Step 2: Trigger the Update**

Apply the updated manifest or set the new container image directly via CLI:

**Command:**

    kubectl set image deployment/myapp-deployment my-app=myregistry/myapp:v2.0.0 -n <namespace>

**Step 3: Monitor the Progress**

Track the rollout status in real time to ensure new Pods become healthy.

**Command:**

    kubectl rollout status deployment/myapp-deployment -n <namespace>

## 2. HOW TO ROLL BACK IF THE DEPLOYMENT FAILS

If the new version crashes, fails readiness checks, or throws errors, Kubernetes allows you to roll back immediately to a previous revision.

**Step 1: Inspect Rollout History**

List all recorded deployment revisions to identify the stable version.

**Command:**

    kubectl rollout history deployment/myapp-deployment -n <namespace>

**Step 2: Undo the Deployment (Roll Back)**

- Roll back to the IMMEDIATELY PREVIOUS version:

      kubectl rollout undo deployment/myapp-deployment -n <namespace>

- Roll back to a SPECIFIC revision number (e.g., Revision 2):

      kubectl rollout undo deployment/myapp-deployment --to-revision=2 -n <namespace>

**Step 3: Verify Rollback Success**

Confirm that all Pods have successfully reverted to the healthy state.

**Command:**

    kubectl rollout status deployment/myapp-deployment -n <namespace>


# "What's the difference between RollingUpdate and Recreate strategy, and when would you use Recreate?" → Recreate kills all old pods before starting new ones — causes downtime, used when old/new versions can't run simultaneously (e.g., DB schema conflicts).

## ROLLINGUPDATE VS. RECREATE STRATEGY

### 1. CORE DEFINITIONS:

- **RollingUpdate** : Replaces old Pods with new Pods gradually. Ensures continuous availability so users experience zero downtime.
- **Recreate** : Kills ALL existing running Pods simultaneously BEFORE starting any new Pods. Guarantees downtime while new Pods boot up.

### 2. KEY DIFFERENCES:

| FEATURE | ROLLINGUPDATE | RECREATE |
|---|---|---|
| Downtime | Zero downtime | Guaranteed downtime |
| Version Coexistence | Old and new versions run concurrently during update | Old and new versions NEVER run at the same time |
| Resource Usage | Requires extra cluster capacity for surge Pods (maxSurge) | No extra capacity needed |
| Rollback Speed | Gradual replacement back | Quick restart back to old |

## WHEN TO USE THE RECREATE STRATEGY

Use Recreate when your system cannot tolerate two different application versions running concurrently:

### 1. BREAKING DATABASE SCHEMA CHANGES:

When an update includes destructive or non-backward-compatible database schema migrations (e.g., dropping columns, changing data types), running old and new app instances at the same time will cause data corruption or app crashes.

### 2. READWRITEONCE (RWO) PERSISTENT VOLUMES:

If a Pod uses a PersistentVolume mounted in ReadWriteOnce mode, only ONE Pod can attach to that storage volume at a time. A RollingUpdate will fail because the new Pod cannot mount the volume until the old Pod terminates.

### 3. SINGLETON PROCESSSES & STATEFUL MONOLITHS:

Legacy services, background batch processors, or stateful applications where running multiple instances simultaneously leads to duplicate task execution or race conditions.

### 4. NON-PRODUCTION ENVIRONMENT RESOURCE SAVINGS:

In dev/test environments with limited CPU/RAM, Recreate avoids allocating extra resources for temporary surge Pods during deployments.

## MANIFEST EXAMPLE

    spec:
      replicas: 3
      strategy:
        type: Recreate   # Kills all 3 old pods first, then spawns 3 new pods

# How does maxSurge and maxUnavailable affect rollout behavior? → maxSurge = extra pods allowed above desired count during update; maxUnavailable = how many can be down at once. Tuning these balances speed vs availability.

## UNDERSTANDING maxSurge AND maxUnavailable

### 1. CORE DEFINITIONS:

In a RollingUpdate deployment strategy, maxSurge and maxUnavailable control how fast Kubernetes replaces old Pods with new Pods and how much capacity is maintained during the rollout.

- **maxSurge** : The MAXIMUM number of Pods that can be created ABOVE the desired replica count during an update.
- **maxUnavailable** : The MAXIMUM number of Pods that can be UN-AVAILABLE (taken offline) below the desired replica count during an update.

**Note:** Both parameters can be set as an absolute number (e.g., 2) or a percentage of target replicas (e.g., 25%).

## HOW THEY CONTROL ROLLOUT BEHAVIOR (EXAMPLE: 4 REPLICAS)

**Target Replicas = 4**

### SCENARIO A: DEFAULT SETTINGS (maxSurge: 25%, maxUnavailable: 25%)

- maxSurge = 25% of 4 = 1 extra Pod (Max allowed total Pods = 5)
- maxUnavailable = 25% of 4 = 1 offline Pod (Min required ready Pods = 3)

**Behavior:**

1. K8s creates 1 new Pod (Total = 5) and terminates 1 old Pod (Active = 3).
2. Waits for new Pods to pass readiness probes.
3. Repeats until all 4 Pods are running the new version.

**Result:** Balanced speed and safety with guaranteed 75% traffic capacity.

# How does Kubernetes know a rollout has failed, and how do you set that up? → Via readiness probes not passing + progressDeadlineSeconds; after that, the rollout is marked failed but won't auto-rollback unless you script it.

## HOW KUBERNETES DETECTS A FAILED ROLLOUT

### 1. HOW KUBERNETES KNOWS A ROLLOUT HAS FAILED:

Kubernetes does NOT automatically consider a deployment failed just because a Pod crashes once. It determines failure through a combination of two mechanisms:

- **Readiness Probes:**
  As new Pods are spawned during a rolling update, Kubernetes runs their readiness probes. If a new Pod crashes, enters CrashLoopBackOff, or fails its readiness check, Kubernetes stops routing traffic to it and halts further rollout progress (it stops replacing the remaining old Pods).

- **progressDeadlineSeconds:**
  This field defines the maximum time (in seconds) Kubernetes will wait for a rollout to make progress. If new Pods fail to reach a "Ready" state within this timeframe, Kubernetes officially marks the Deployment status as Progressing = False (Reason: ProgressDeadlineExceeded).

### 2. IMPORTANT CAVEAT (NO AUTOMATIC ROLLBACK):

When progressDeadlineSeconds is exceeded:

- Kubernetes MARKS the deployment as failed.
- Kubernetes STOPS creating new broken Pods.
- Kubernetes DOES NOT automatically roll back to the previous revision by default.
- You must trigger a rollback via CLI, CI/CD pipeline, or GitOps controller.

## HOW TO SET UP FAILURE DETECTION IN YOUR DEPLOYMENT MANIFEST

Add `progressDeadlineSeconds` under `spec`, and configure robust health probes under the container spec.

### Example Deployment Manifest:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 4
      progressDeadlineSeconds: 300   # Fail rollout if not complete in 5 minutes (300s)
      revisionHistoryLimit: 10       # Retain past 10 revisions for rollback
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxSurge: 25%
          maxUnavailable: 25%
      template:
        spec:
          containers:
          - name: my-app
            image: myregistry/myapp:v2.0.0
            readinessProbe:
              httpGet:
                path: /healthz
                port: 8080
              initialDelaySeconds: 10
              periodSeconds: 5
              failureThreshold: 3    # Mark unready after 3 failures

## AUTOMATING ROLLBACKS IN CI/CD OR GITOPS

Since Kubernetes natively halts rather than auto-reverts, you can automate rollbacks in your deployment workflow:

### Option A: CI/CD Pipeline Scripting (e.g., GitHub Actions / Jenkins)

After triggering a deployment, monitor `kubectl rollout status`. If it times out, trigger an automatic rollback:

### Command Script:

    # Monitor status with a timeout
    if ! kubectl rollout status deployment/my-app --timeout=300s -n <namespace>; then
      echo "Rollout failed! Initiating automatic rollback..."
      kubectl rollout undo deployment/my-app -n <namespace>
      exit 1
    fi

### Option B: GitOps Operators (e.g., Argo Rollouts / Flagger)

Advanced progressive delivery tools like Argo Rollouts or Flagger natively extend Kubernetes to perform automated rollbacks based on real-time metrics (e.g., HTTP 5xx error rates or latency spikes measured via Prometheus/Grafana).

# "If a rollback also fails, what's your next step?" → Check revision history (kubectl rollout history), roll back to a specific known-good revision, or manually redeploy a pinned image tag.

## WHAT TO DO IF A KUBERNETES ROLLBACK FAILS

If running `kubectl rollout undo` fails or reverts to a state that is also broken, it usually means the immediately preceding revision was corrupted, missing secrets, or pointing to a deprecated container image tag.

## STEP-BY-STEP RECOVERY WORKFLOW

### STEP 1: INSPECT REVISION HISTORY

List all saved Deployment revisions to identify a confirmed, known-good historical revision (rather than relying on the default n-1 rollback target).

**Command:**

    kubectl rollout history deployment/<deployment-name> -n <namespace>

Inspect details of a specific older revision:

    kubectl rollout history deployment/<deployment-name> --revision=<number> -n <namespace>

### STEP 2: ROLL BACK TO A SPECIFIC KNOWN-GOOD REVISION

Target an explicit, verified revision number from your deployment history.

**Command:**

    kubectl rollout undo deployment/<deployment-name> --to-revision=<number> -n <namespace>

### STEP 3: MANUALLY FORCE A PINNED STABLE CONTAINER IMAGE

If the revision history is corrupted or unavailable, override the deployment image directly using an explicitly tagged, stable production image (e.g., `v1.8.0` instead of `latest`).

**Command:**

    kubectl set image deployment/<deployment-name> <container-name>=<registry>/<image>:v1.8.0 -n <namespace>

### STEP 4: PAUSE ROLLOUT & FIX CLUSTER-LEVEL DEPENDENCIES

If the rollback is failing due to cluster-level dependencies (e.g., a missing ConfigMap, deleted Secret, or database migration failure):

1. Pause the deployment to stop the continuous crash/restart churn:

       kubectl rollout pause deployment/<deployment-name> -n <namespace>

2. Re-create or fix the missing cluster resources (Secrets, ConfigMaps, PVCs).

3. Resume the deployment once dependencies are restored:

       kubectl rollout resume deployment/<deployment-name> -n <namespace>

