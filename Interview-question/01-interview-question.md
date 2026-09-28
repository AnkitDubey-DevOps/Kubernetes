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
