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
