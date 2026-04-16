# Sandbox Providers Guide

Open SWE runs every agent task in an isolated **sandbox** — a remote Linux environment with full shell access where the repo is cloned, code is written, and commands are executed. The sandbox is the agent's workspace; nothing it does can affect production systems or other tasks.

This guide covers the built-in sandbox providers, how to add your own (AWS ECS, Docker, Kubernetes, etc.), and production considerations.

> **Quick start:** If you just want to change which built-in provider you use, set the `SANDBOX_TYPE` environment variable. See [Built-in providers](#built-in-providers) below.

---

## Architecture overview

```
┌─────────────────────────────────────────────────┐
│  Open SWE Agent (LangGraph)                     │
│                                                 │
│  create_deep_agent(                             │
│      backend=sandbox_backend,  ◄── any object   │
│      ...                           implementing │
│  )                                 the protocol │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│  SandboxBackendProtocol                         │
│                                                 │
│  • execute(command, timeout) -> ExecuteResponse  │
│  • id -> str                                    │
│  • ls(), read(), write(), edit(), glob(), grep() │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┼────────────┬──────────────┐
        ▼            ▼            ▼              ▼
   LangSmith      Modal      Daytona     Your provider
   sandbox       sandbox     sandbox     (ECS, K8s, Docker, …)
```

The agent interacts with the sandbox exclusively through `SandboxBackendProtocol` from the `deepagents` package. Any backend that implements this protocol can be plugged in.

### SandboxBackendProtocol

The protocol requires:

| Method / Property | Purpose |
|---|---|
| `id` (property) | Returns a unique string identifier for the sandbox instance |
| `execute(command, *, timeout=None) -> ExecuteResponse` | Runs a shell command and returns stdout+stderr, exit code, and truncation flag |
| `ls(path)` | List directory contents |
| `read(path)` | Read file contents |
| `write(path, content)` | Write file contents |
| `edit(path, edits)` | Apply edits to a file |
| `glob(pattern)` | Find files matching a glob pattern |
| `grep(pattern, path)` | Search file contents |

### BaseSandbox shortcut

`deepagents.backends.sandbox.BaseSandbox` implements all file operations (`ls`, `read`, `write`, `edit`, `glob`, `grep`) by delegating to `execute()`. If your provider can run shell commands, extend `BaseSandbox` and only implement `execute()` and the `id` property:

```python
from deepagents.backends.sandbox import BaseSandbox
from deepagents.backends.protocol import ExecuteResponse

class MySandbox(BaseSandbox):
    def __init__(self, connection):
        self._conn = connection

    @property
    def id(self) -> str:
        return self._conn.id

    def execute(self, command: str, *, timeout: int | None = None) -> ExecuteResponse:
        result = self._conn.run(command, timeout=timeout or 300)
        return ExecuteResponse(
            output=result.stdout + result.stderr,
            exit_code=result.exit_code,
            truncated=False,
        )
```

### Factory + registration pattern

Each provider is a factory function registered in `agent/utils/sandbox.py`:

```python
# agent/utils/sandbox.py
SANDBOX_FACTORIES = {
    "langsmith": create_langsmith_sandbox,
    "daytona": create_daytona_sandbox,
    "modal": create_modal_sandbox,
    "runloop": create_runloop_sandbox,
    "local": create_local_sandbox,
}
```

The active provider is selected by the `SANDBOX_TYPE` environment variable (defaults to `langsmith`). Your factory function receives an optional `sandbox_id` for reconnecting to an existing sandbox:

```python
def create_my_sandbox(sandbox_id: str | None = None):
    if sandbox_id:
        # Reconnect to existing sandbox
        ...
    else:
        # Create a new sandbox
        ...
    return my_sandbox_backend  # implements SandboxBackendProtocol
```

---

## Built-in providers

### LangSmith (default)

[LangSmith](https://smith.langchain.com/) provides managed cloud sandboxes with built-in tracing.

```bash
SANDBOX_TYPE="langsmith"                        # default — can be omitted
LANGSMITH_API_KEY_PROD="lsv2_..."               # required
DEFAULT_SANDBOX_TEMPLATE_NAME="my-template"     # optional custom template
DEFAULT_SANDBOX_TEMPLATE_IMAGE="my-org/img:v1"  # optional custom Docker image
```

**Pros:** Zero infrastructure to manage, integrated tracing and observability, proxy-based GitHub auth.
**Cons:** Requires a LangSmith account, sandbox quotas apply.

See [INSTALLATION.md § 4c](../INSTALLATION.md#4c-sandbox-templates-optional) for template setup.

### Modal

[Modal](https://modal.com/) runs sandboxes as serverless containers.

```bash
SANDBOX_TYPE="modal"
MODAL_APP_NAME="open-swe"        # optional, defaults to "open-swe"
# Modal credentials are configured via `modal token set` or environment variables
```

**Pros:** Fast cold starts, automatic scaling, pay-per-second billing.
**Cons:** Requires a Modal account.

### Daytona

[Daytona](https://www.daytona.io/) provides full development environments.

```bash
SANDBOX_TYPE="daytona"
DAYTONA_API_KEY="dtna_..."       # required
```

**Pros:** Full dev environment with IDE support, persistent workspaces.
**Cons:** Requires a Daytona account.

### Runloop

[Runloop](https://www.runloop.ai/) provides cloud devbox environments.

```bash
SANDBOX_TYPE="runloop"
RUNLOOP_API_KEY="rl_..."         # required
```

**Pros:** Purpose-built for AI coding agents, fast provisioning.
**Cons:** Requires a Runloop account.

### Local (development only)

Runs commands directly on your host machine with **no isolation**.

```bash
SANDBOX_TYPE="local"
LOCAL_SANDBOX_ROOT_DIR="/tmp/open-swe-sandbox"  # optional, defaults to cwd
```

> ⚠️ **Warning:** No sandboxing. The agent executes commands directly on your machine. Only use for local development with human-in-the-loop review.

---

## Adding a custom sandbox provider

This section walks through implementing sandbox providers for three common infrastructure choices: AWS ECS, Docker (local or remote), and Kubernetes. Each follows the same pattern:

1. Create an integration file at `agent/integrations/<provider>.py`
2. Implement a sandbox class extending `BaseSandbox`
3. Implement a factory function
4. Register it in `agent/utils/sandbox.py`

---

### AWS ECS provider

Run each sandbox as a [Fargate](https://aws.amazon.com/fargate/) task in your own AWS account. This gives you full control over networking, IAM, and compute.

#### Prerequisites

- An ECS cluster (Fargate launch type)
- A task definition using your sandbox Docker image (see the project [Dockerfile](../Dockerfile))
- A VPC with subnets and security groups allowing outbound internet (for `git clone`, package installs)
- AWS credentials available to the Open SWE process (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` or IAM role)
- `boto3` installed (`pip install boto3`)

#### Integration file

```python
# agent/integrations/ecs.py
import os
import time
import uuid
import logging

import boto3
from deepagents.backends.sandbox import BaseSandbox
from deepagents.backends.protocol import ExecuteResponse

logger = logging.getLogger(__name__)


class ECSSandbox(BaseSandbox):
    """Sandbox backed by an AWS ECS Fargate task with SSM exec."""

    def __init__(self, cluster: str, task_arn: str, container_name: str, region: str):
        self._cluster = cluster
        self._task_arn = task_arn
        self._container_name = container_name
        self._ssm = boto3.client("ssm", region_name=region)
        self._ecs = boto3.client("ecs", region_name=region)

    @property
    def id(self) -> str:
        # Extract task ID from ARN
        return self._task_arn.rsplit("/", 1)[-1]

    def execute(self, command: str, *, timeout: int | None = None) -> ExecuteResponse:
        """Execute a command in the ECS task via ECS Exec (SSM).

        NOTE: This is a skeleton — you must implement _run_ssm_command()
        to stream output from the SSM session. See the "Production note
        on ECS Exec" section below for concrete approaches.
        """
        timeout = timeout or 300
        try:
            # Wrap command to capture exit code reliably
            wrapped = f'{command}\nEXIT_CODE=$?\necho "___EXIT_CODE:$EXIT_CODE"\nexit $EXIT_CODE'
            output, exit_code = self._run_ssm_command(wrapped, timeout)
            return ExecuteResponse(
                output=output,
                exit_code=exit_code,
                truncated=False,
            )
        except Exception as e:
            return ExecuteResponse(
                output=str(e),
                exit_code=1,
                truncated=False,
            )

    def _run_ssm_command(self, command: str, timeout: int) -> tuple[str, int]:
        """Run a command via ECS Exec and return (output, exit_code).

        PLACEHOLDER — replace with your SSM session implementation.

        Production options:
        1. Use the session-manager-plugin to open a websocket session.
        2. Run an HTTP command server inside the container instead.
        3. SSH into the container's ENI IP.

        See: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html
        """
        raise NotImplementedError(
            "Implement _run_ssm_command() — see docstring for options."
        )


def create_ecs_sandbox(sandbox_id: str | None = None):
    """Create or reconnect to an ECS Fargate sandbox.

    Required environment variables:
        ECS_CLUSTER: ECS cluster name or ARN
        ECS_TASK_DEFINITION: Task definition family:revision or ARN
        ECS_SUBNETS: Comma-separated subnet IDs
        ECS_SECURITY_GROUPS: Comma-separated security group IDs
        ECS_CONTAINER_NAME: Container name in the task definition (default: "sandbox")
        AWS_DEFAULT_REGION: AWS region (default: "us-east-1")

    Args:
        sandbox_id: Optional existing task ARN to reconnect to.

    Returns:
        ECSSandbox implementing SandboxBackendProtocol.
    """
    cluster = os.environ["ECS_CLUSTER"]
    container_name = os.getenv("ECS_CONTAINER_NAME", "sandbox")
    region = os.getenv("AWS_DEFAULT_REGION", "us-east-1")
    ecs = boto3.client("ecs", region_name=region)

    if sandbox_id:
        # Reconnect to existing task
        task_arn = sandbox_id
        response = ecs.describe_tasks(cluster=cluster, tasks=[task_arn])
        if not response["tasks"] or response["tasks"][0]["lastStatus"] != "RUNNING":
            raise RuntimeError(f"ECS task {task_arn} is not running")
    else:
        # Launch a new Fargate task
        task_definition = os.environ["ECS_TASK_DEFINITION"]
        subnets = os.environ["ECS_SUBNETS"].split(",")
        security_groups = os.environ["ECS_SECURITY_GROUPS"].split(",")

        response = ecs.run_task(
            cluster=cluster,
            taskDefinition=task_definition,
            launchType="FARGATE",
            enableExecuteCommand=True,  # Required for ECS Exec
            networkConfiguration={
                "awsvpcConfiguration": {
                    "subnets": subnets,
                    "securityGroups": security_groups,
                    "assignPublicIp": "ENABLED",
                }
            },
            count=1,
        )
        task_arn = response["tasks"][0]["taskArn"]

        # Wait for task to reach RUNNING state
        waiter = ecs.get_waiter("tasks_running")
        waiter.wait(cluster=cluster, tasks=[task_arn])
        logger.info("ECS sandbox task running: %s", task_arn)

    return ECSSandbox(
        cluster=cluster,
        task_arn=task_arn,
        container_name=container_name,
        region=region,
    )
```

#### Environment variables

```bash
SANDBOX_TYPE="ecs"
ECS_CLUSTER="open-swe-sandboxes"
ECS_TASK_DEFINITION="open-swe-sandbox:1"
ECS_SUBNETS="subnet-abc123,subnet-def456"
ECS_SECURITY_GROUPS="sg-abc123"
ECS_CONTAINER_NAME="sandbox"          # optional, defaults to "sandbox"
AWS_DEFAULT_REGION="us-east-1"        # optional, defaults to "us-east-1"
```

#### ECS task definition tips

- Use the project [Dockerfile](../Dockerfile) as your container image (or a custom image based on it).
- Enable ECS Exec in the task definition (`enableExecuteCommand: true`) so the agent can run commands via SSM.
- Set a generous `stopTimeout` (e.g. 120s) to allow graceful cleanup.
- Assign an IAM task role with **no** production permissions — the sandbox should only be able to reach the internet for git operations and package installs.
- Consider using `ephemeralStorage` for larger repos.

#### Production note on ECS Exec

The example above uses [ECS Exec](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html) (backed by SSM) to run commands inside the task. In production, you have several options for command execution:

1. **ECS Exec + SSM session plugin** — Full interactive session support. Requires the [session-manager-plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) or a websocket implementation.
2. **SSH into the container** — Install an SSH server in your sandbox image and connect over the task's ENI IP.
3. **HTTP command API** — Run a lightweight HTTP server in the container that accepts command execution requests. This is the simplest approach and avoids SSM complexity.

---

### Docker provider

Run sandboxes as local Docker containers. Useful for self-hosted deployments, CI environments, or air-gapped networks where external sandbox services are unavailable.

#### Prerequisites

- Docker Engine installed and the Docker daemon running
- The Docker socket accessible to the Open SWE process (`/var/run/docker.sock`)
- `docker` Python SDK installed (`pip install docker`)
- A sandbox image built from the project [Dockerfile](../Dockerfile) or your own

#### Integration file

```python
# agent/integrations/docker_sandbox.py
import os
import logging

import docker
from deepagents.backends.sandbox import BaseSandbox
from deepagents.backends.protocol import ExecuteResponse

logger = logging.getLogger(__name__)

DEFAULT_DOCKER_IMAGE = "open-swe-sandbox:latest"


class DockerSandbox(BaseSandbox):
    """Sandbox backed by a local Docker container."""

    def __init__(self, container):
        self._container = container

    @property
    def id(self) -> str:
        return self._container.id

    def execute(self, command: str, *, timeout: int | None = None) -> ExecuteResponse:
        """Execute a command inside the Docker container."""
        timeout = timeout or 300
        try:
            exec_result = self._container.exec_run(
                cmd=["bash", "-c", command],
                workdir="/workspace",
                demux=True,
            )
            stdout = (exec_result.output[0] or b"").decode("utf-8", errors="replace")
            stderr = (exec_result.output[1] or b"").decode("utf-8", errors="replace")
            return ExecuteResponse(
                output=stdout + stderr,
                exit_code=exec_result.exit_code,
                truncated=False,
            )
        except Exception as e:
            return ExecuteResponse(
                output=str(e),
                exit_code=1,
                truncated=False,
            )


def create_docker_sandbox(sandbox_id: str | None = None):
    """Create or reconnect to a Docker container sandbox.

    Required environment variables:
        DOCKER_SANDBOX_IMAGE: Docker image to use (default: "open-swe-sandbox:latest")

    Optional environment variables:
        DOCKER_SANDBOX_MEMORY: Memory limit (e.g. "4g", default: "4g")
        DOCKER_SANDBOX_CPUS: CPU limit (e.g. "2.0", default: "2.0")
        DOCKER_SANDBOX_NETWORK: Docker network to attach (default: "bridge")

    Args:
        sandbox_id: Optional existing container ID to reconnect to.

    Returns:
        DockerSandbox implementing SandboxBackendProtocol.
    """
    client = docker.from_env()
    image = os.getenv("DOCKER_SANDBOX_IMAGE", DEFAULT_DOCKER_IMAGE)
    memory = os.getenv("DOCKER_SANDBOX_MEMORY", "4g")
    cpus = float(os.getenv("DOCKER_SANDBOX_CPUS", "2.0"))
    network = os.getenv("DOCKER_SANDBOX_NETWORK", "bridge")

    if sandbox_id:
        try:
            container = client.containers.get(sandbox_id)
            if container.status != "running":
                container.start()
            logger.info("Reconnected to Docker sandbox: %s", sandbox_id)
        except docker.errors.NotFound:
            raise RuntimeError(f"Docker container {sandbox_id} not found")
    else:
        container = client.containers.run(
            image=image,
            command="sleep infinity",   # Keep container alive
            detach=True,
            remove=False,               # Don't auto-remove — we manage lifecycle
            working_dir="/workspace",
            mem_limit=memory,
            nano_cpus=int(cpus * 1e9),
            network=network,
            labels={"managed-by": "open-swe"},
        )
        logger.info("Created Docker sandbox: %s", container.id)

    return DockerSandbox(container)
```

#### Environment variables

```bash
SANDBOX_TYPE="docker"
DOCKER_SANDBOX_IMAGE="open-swe-sandbox:latest"  # optional
DOCKER_SANDBOX_MEMORY="4g"                       # optional
DOCKER_SANDBOX_CPUS="2.0"                        # optional
DOCKER_SANDBOX_NETWORK="bridge"                  # optional
```

#### Build the sandbox image

```bash
# From the open-swe project root
docker build -t open-swe-sandbox:latest .
```

#### Docker-in-Docker considerations

If Open SWE itself runs inside a container (e.g. in a CI pipeline), you need to share the Docker socket:

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock open-swe-server
```

Or use a [Docker-in-Docker (DinD)](https://hub.docker.com/_/docker) sidecar.

---

### Kubernetes provider

Run each sandbox as a [Kubernetes Pod](https://kubernetes.io/docs/concepts/workloads/pods/). This is ideal for organizations that already manage a Kubernetes cluster and want to leverage existing infrastructure, RBAC, network policies, and resource quotas.

#### Prerequisites

- A Kubernetes cluster (EKS, GKE, AKS, or self-hosted)
- `kubectl` access from the Open SWE process (in-cluster ServiceAccount or kubeconfig)
- `kubernetes` Python client installed (`pip install kubernetes`)
- A sandbox image pushed to a registry accessible from the cluster
- A dedicated namespace for sandbox pods (recommended)

#### Integration file

```python
# agent/integrations/kubernetes_sandbox.py
import os
import uuid
import logging
import time

from kubernetes import client, config as k8s_config
from kubernetes.stream import stream
from deepagents.backends.sandbox import BaseSandbox
from deepagents.backends.protocol import ExecuteResponse

logger = logging.getLogger(__name__)

DEFAULT_K8S_IMAGE = "open-swe-sandbox:latest"
DEFAULT_K8S_NAMESPACE = "open-swe-sandboxes"


class KubernetesSandbox(BaseSandbox):
    """Sandbox backed by a Kubernetes Pod."""

    def __init__(self, pod_name: str, namespace: str, core_api: client.CoreV1Api):
        self._pod_name = pod_name
        self._namespace = namespace
        self._api = core_api

    @property
    def id(self) -> str:
        return self._pod_name

    def execute(self, command: str, *, timeout: int | None = None) -> ExecuteResponse:
        """Execute a command inside the Kubernetes pod via exec."""
        timeout = timeout or 300
        try:
            # Wrap command to capture the real exit code on the last line
            wrapped = f'{command}\necho ""\necho "___EXIT_CODE:$?"'
            resp = stream(
                self._api.connect_get_namespaced_pod_exec,
                self._pod_name,
                self._namespace,
                command=["bash", "-c", wrapped],
                container="sandbox",
                stderr=True,
                stdout=True,
                stdin=False,
                tty=False,
                _request_timeout=timeout,
            )
            # Parse exit code from the last line
            output = resp
            exit_code = 0
            if "___EXIT_CODE:" in output:
                lines = output.rsplit("___EXIT_CODE:", 1)
                output = lines[0]
                try:
                    exit_code = int(lines[1].strip())
                except ValueError:
                    pass
            return ExecuteResponse(
                output=output,
                exit_code=exit_code,
                truncated=False,
            )
        except Exception as e:
            return ExecuteResponse(
                output=str(e),
                exit_code=1,
                truncated=False,
            )


def create_kubernetes_sandbox(sandbox_id: str | None = None):
    """Create or reconnect to a Kubernetes Pod sandbox.

    Required environment variables:
        K8S_SANDBOX_NAMESPACE: Namespace for sandbox pods (default: "open-swe-sandboxes")
        K8S_SANDBOX_IMAGE: Container image (default: "open-swe-sandbox:latest")

    Optional environment variables:
        K8S_SANDBOX_CPU_REQUEST: CPU request (default: "500m")
        K8S_SANDBOX_CPU_LIMIT: CPU limit (default: "2")
        K8S_SANDBOX_MEMORY_REQUEST: Memory request (default: "1Gi")
        K8S_SANDBOX_MEMORY_LIMIT: Memory limit (default: "4Gi")
        K8S_SANDBOX_SERVICE_ACCOUNT: ServiceAccount for sandbox pods
        K8S_SANDBOX_NODE_SELECTOR: JSON node selector (e.g. '{"sandbox": "true"}')

    Args:
        sandbox_id: Optional existing pod name to reconnect to.

    Returns:
        KubernetesSandbox implementing SandboxBackendProtocol.
    """
    # Load kubeconfig: in-cluster if available, else from ~/.kube/config
    try:
        k8s_config.load_incluster_config()
    except k8s_config.ConfigException:
        k8s_config.load_kube_config()

    core_api = client.CoreV1Api()
    namespace = os.getenv("K8S_SANDBOX_NAMESPACE", DEFAULT_K8S_NAMESPACE)
    image = os.getenv("K8S_SANDBOX_IMAGE", DEFAULT_K8S_IMAGE)

    if sandbox_id:
        # Reconnect to existing pod
        pod = core_api.read_namespaced_pod(name=sandbox_id, namespace=namespace)
        if pod.status.phase != "Running":
            raise RuntimeError(f"Pod {sandbox_id} is in phase {pod.status.phase}, expected Running")
        logger.info("Reconnected to Kubernetes sandbox pod: %s", sandbox_id)
        return KubernetesSandbox(sandbox_id, namespace, core_api)

    # Create a new sandbox pod
    pod_name = f"open-swe-sandbox-{uuid.uuid4().hex[:8]}"

    cpu_request = os.getenv("K8S_SANDBOX_CPU_REQUEST", "500m")
    cpu_limit = os.getenv("K8S_SANDBOX_CPU_LIMIT", "2")
    mem_request = os.getenv("K8S_SANDBOX_MEMORY_REQUEST", "1Gi")
    mem_limit = os.getenv("K8S_SANDBOX_MEMORY_LIMIT", "4Gi")
    service_account = os.getenv("K8S_SANDBOX_SERVICE_ACCOUNT")

    pod_manifest = client.V1Pod(
        metadata=client.V1ObjectMeta(
            name=pod_name,
            namespace=namespace,
            labels={
                "app": "open-swe-sandbox",
                "managed-by": "open-swe",
            },
        ),
        spec=client.V1PodSpec(
            containers=[
                client.V1Container(
                    name="sandbox",
                    image=image,
                    command=["sleep", "infinity"],
                    working_dir="/workspace",
                    resources=client.V1ResourceRequirements(
                        requests={"cpu": cpu_request, "memory": mem_request},
                        limits={"cpu": cpu_limit, "memory": mem_limit},
                    ),
                )
            ],
            restart_policy="Never",
            service_account_name=service_account,
            # Sandbox pods should not access the Kubernetes API
            automount_service_account_token=False,
        ),
    )

    core_api.create_namespaced_pod(namespace=namespace, body=pod_manifest)

    # Wait for pod to be Running
    for _ in range(120):
        pod = core_api.read_namespaced_pod(name=pod_name, namespace=namespace)
        if pod.status.phase == "Running":
            break
        time.sleep(1)
    else:
        # Cleanup on timeout
        core_api.delete_namespaced_pod(name=pod_name, namespace=namespace)
        raise RuntimeError(f"Pod {pod_name} did not reach Running state within 120 seconds")

    logger.info("Created Kubernetes sandbox pod: %s", pod_name)
    return KubernetesSandbox(pod_name, namespace, core_api)
```

#### Environment variables

```bash
SANDBOX_TYPE="kubernetes"
K8S_SANDBOX_NAMESPACE="open-swe-sandboxes"       # optional
K8S_SANDBOX_IMAGE="my-registry/open-swe:latest"  # optional
K8S_SANDBOX_CPU_REQUEST="500m"                    # optional
K8S_SANDBOX_CPU_LIMIT="2"                         # optional
K8S_SANDBOX_MEMORY_REQUEST="1Gi"                  # optional
K8S_SANDBOX_MEMORY_LIMIT="4Gi"                    # optional
K8S_SANDBOX_SERVICE_ACCOUNT="sandbox-sa"          # optional
```

#### Kubernetes setup recommendations

1. **Dedicated namespace:** Create a `open-swe-sandboxes` namespace with resource quotas to limit total sandbox resource usage.

2. **RBAC:** The Open SWE controller needs permission to create, get, delete pods and exec into them. Create a minimal ClusterRole:

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: open-swe-sandbox-manager
      namespace: open-swe-sandboxes
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["create", "get", "list", "delete"]
    - apiGroups: [""]
      resources: ["pods/exec"]
      verbs: ["create"]
    ```

3. **Network policies:** Restrict sandbox pod egress to only what's needed (GitHub, package registries). Deny access to the Kubernetes API server and internal services.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: sandbox-egress
      namespace: open-swe-sandboxes
    spec:
      podSelector:
        matchLabels:
          app: open-swe-sandbox
      policyTypes:
      - Egress
      egress:
      - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
            - 10.0.0.0/8        # Block cluster-internal traffic
            - 172.16.0.0/12
            - 192.168.0.0/16
        ports:
        - protocol: TCP
          port: 443             # HTTPS (git, package registries)
        - protocol: TCP
          port: 80              # HTTP fallback
    ```

4. **Pod cleanup:** Use a TTL controller or CronJob to garbage-collect sandbox pods that have been running longer than a threshold (e.g. 2 hours). Add the `managed-by: open-swe` label selector for safe cleanup:

    ```bash
    kubectl delete pods -n open-swe-sandboxes -l managed-by=open-swe \
        --field-selector=status.phase=Running \
        --grace-period=30
    ```

5. **Node isolation (optional):** Use taints and tolerations to schedule sandbox pods onto dedicated nodes, preventing resource contention with other workloads.

---

## Registering a custom provider

After creating your integration file, register it in `agent/utils/sandbox.py`:

```python
# agent/utils/sandbox.py
from agent.integrations.ecs import create_ecs_sandbox
from agent.integrations.docker_sandbox import create_docker_sandbox
from agent.integrations.kubernetes_sandbox import create_kubernetes_sandbox

SANDBOX_FACTORIES = {
    "langsmith": create_langsmith_sandbox,
    "daytona": create_daytona_sandbox,
    "modal": create_modal_sandbox,
    "runloop": create_runloop_sandbox,
    "local": create_local_sandbox,
    # Custom providers
    "ecs": create_ecs_sandbox,
    "docker": create_docker_sandbox,
    "kubernetes": create_kubernetes_sandbox,
}
```

Then set `SANDBOX_TYPE` to your provider name:

```bash
SANDBOX_TYPE="docker"      # or "ecs", "kubernetes", etc.
```

---

## Production considerations

### Sandbox lifecycle

Open SWE reuses sandboxes across follow-up messages on the same thread. Your provider should support:

- **Create** — Spin up a new sandbox for a new thread.
- **Reconnect** — Reconnect to an existing sandbox by ID when the agent is re-invoked.
- **Health check** — The agent pings the sandbox with `echo ok` before each use. If the sandbox is unreachable, it automatically recreates one via `_recreate_sandbox()` in `agent/server.py`.
- **Cleanup** — Implement a cleanup strategy (TTL, idle timeout, or explicit deletion) to reclaim resources from finished tasks.

### Security

- **No production credentials** inside the sandbox. The sandbox should only have access to git operations and package registries.
- **Read-only root filesystem** where possible — mount `/workspace` as a writable volume.
- **Drop all Linux capabilities** except what's needed for shell execution.
- **Disable privilege escalation** (`allowPrivilegeEscalation: false` in Kubernetes, `--security-opt=no-new-privileges` in Docker).
- **Run as non-root** if your sandbox image supports it.

### Networking

- Allow outbound HTTPS (port 443) for GitHub, package registries, and LLM APIs.
- Block access to internal services, metadata endpoints (e.g. `169.254.169.254` on AWS), and the orchestration layer.
- Consider using a forward proxy for logging and allowlisting outbound destinations.

### Resource limits

- Set CPU and memory limits to prevent a single sandbox from consuming excessive resources.
- Set ephemeral storage limits for disk-intensive repos.
- Implement a maximum sandbox lifetime (e.g. 2 hours) as a safety net.

### GitHub authentication in custom sandboxes

The default LangSmith provider uses a proxy to inject GitHub credentials. For custom providers, you have two options:

1. **Proxy-based** — Run a forward proxy that injects auth headers (same approach as LangSmith). This keeps credentials out of the sandbox entirely.
2. **Token injection** — Pass the GitHub token into the sandbox as an environment variable and configure git to use it:
    ```bash
    git config --global url."https://x-access-token:${GITHUB_TOKEN}@github.com/".insteadOf "https://github.com/"
    ```
    This is simpler but means the token is visible inside the sandbox.

---

## Summary

| Provider | Type | Isolation | Best for |
|---|---|---|---|
| **LangSmith** | Managed cloud | Full | Default — zero infrastructure setup |
| **Modal** | Serverless cloud | Full | Fast scaling, pay-per-use |
| **Daytona** | Managed cloud | Full | Full dev environments |
| **Runloop** | Managed cloud | Full | AI agent-optimized |
| **Docker** | Self-hosted | Container | Self-hosted deployments, CI, air-gapped |
| **AWS ECS** | Self-hosted cloud | Container | AWS-native organizations |
| **Kubernetes** | Self-hosted cloud | Pod | Organizations with existing K8s clusters |
| **Local** | None | None | Development only |
