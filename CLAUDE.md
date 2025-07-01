# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Volcano is a Kubernetes-native batch scheduling system that extends the standard kube-scheduler for complex workloads like AI/ML, HPC, and big data applications. It provides advanced scheduling features including gang scheduling, fair resource sharing, preemption, and topology awareness.

## Development Commands

### Build and Testing
```bash
# Build all components
make all

# Build individual components
make vc-scheduler
make vc-controller-manager
make vc-webhook-manager
make vc-agent
make vcctl

# Run unit tests
make unit-test

# Run tests for specific package
go test ./pkg/scheduler/api/...
go test -race ./pkg/controllers/job/...

# Run specific E2E test suites
make e2e-test-schedulingbase
make e2e-test-schedulingaction
make e2e-test-jobp
make e2e-test-jobseq
make e2e-test-vcctl

# Run all E2E tests (requires Docker/Kind)
make e2e

# Build container images
make images

# Generate Kubernetes manifests and CRDs
make generate-yaml
make manifests
```

### Code Quality
```bash
# Run linting
make lint

# Verify code formatting and generated code
make verify

# Update generated code (when modifying APIs)
make generate-code

# License checking
make lint-licenses
make mirror-licenses
```

### Local Development
```bash
# Start local Volcano cluster
./hack/local-up-volcano.sh

# Install Volcano via YAML
kubectl apply -f https://raw.githubusercontent.com/volcano-sh/volcano/master/installer/volcano-development.yaml

# Install via Helm
helm repo add volcano-sh https://volcano-sh.github.io/helm-charts
helm install volcano volcano-sh/volcano -n volcano-system --create-namespace
```

## High-Level Architecture

### Main Components

1. **Scheduler** (`cmd/scheduler/`) - Core scheduling engine using session-based architecture
2. **Controller Manager** (`cmd/controller-manager/`) - Manages Jobs, Queues, PodGroups lifecycle
3. **Webhook Manager** (`cmd/webhook-manager/`) - Admission control for resource validation/mutation
4. **Agent** (`cmd/agent/`) - Node-level resource monitoring and QoS management
5. **CLI Tools** (`cmd/cli/`) - Command-line interface (vcctl)

### Scheduler Framework

The scheduler follows a sophisticated **action-plugin architecture**:

- **Sessions** (`pkg/scheduler/framework/session.go`) - Each scheduling cycle creates a consistent cluster state snapshot
- **Actions** (`pkg/scheduler/actions/`) - Define what to do: allocate, reclaim, preempt, enqueue, backfill, shuffle
- **Plugins** (`pkg/scheduler/plugins/`) - Define how to do it: gang, drf, predicates, priority, capacity, etc.
- **Cache** (`pkg/scheduler/cache/`) - Maintains cluster state for consistent scheduling decisions

**Execution Pattern:**
```
OpenSession() → Execute Actions (using plugins) → CloseSession()
```

**Plugin Tiers:** Plugins execute in ordered tiers for proper scheduling flow:
```yaml
tiers:
  - plugins: [gang, drf]      # Tier 1: Basic scheduling policies
  - plugins: [predicates]     # Tier 2: Node filtering
  - plugins: [nodeorder]      # Tier 3: Node scoring
```

### Controller Architecture

Controllers use **state machine patterns** for resource lifecycle management:

- **Job Controller** (`pkg/controllers/job/`) - Job states: Pending → Running → Completing → Completed/Failed
- **Queue Controller** (`pkg/controllers/queue/`) - Queue states: Open → Closed → Unknown  
- **PodGroup Controller** (`pkg/controllers/podgroup/`) - Manages gang scheduling coordination
- **JobFlow/JobTemplate Controllers** - Workflow and template management
- **HyperNode Controller** - Topology-aware node grouping

### API Layer

Core abstractions in `pkg/scheduler/api/`:
- **JobInfo/TaskInfo** - Represents batch jobs and their constituent tasks
- **NodeInfo** - Node resources and allocation state
- **QueueInfo** - Resource quotas and priority management
- **ClusterInfo** - Complete cluster state snapshot
- **Resource** - Unified resource representation

### Key Architectural Patterns

1. **Session-Based Scheduling** - Discrete scheduling cycles with consistent snapshots
2. **Plugin Architecture** - Extensible behavior through callback-based plugins
3. **Event-Driven Controllers** - Kubernetes informers with work queue processing
4. **State Machine Management** - Clear resource lifecycle with state transitions
5. **Cache-First Design** - Scheduler maintains optimized cluster state cache

## Project Structure

```
cmd/                    # Main entry points
├── scheduler/          # Core scheduling engine
├── controller-manager/ # Resource lifecycle management
├── webhook-manager/    # Admission control
├── agent/             # Node-level agent
└── cli/               # Command-line tools

pkg/
├── scheduler/         # Scheduler framework, actions, plugins
├── controllers/       # Resource controllers with state machines
├── webhooks/          # Admission webhooks
├── cli/              # CLI command implementations
└── agent/            # Agent functionality

config/crd/           # Custom Resource Definitions
installer/            # Deployment manifests and Helm charts
test/e2e/            # End-to-end test suites
docs/                # Design documents and user guides
```

## Common Workflows

### Adding New Scheduler Plugin
1. Create plugin in `pkg/scheduler/plugins/`
2. Implement required interfaces (OnSessionOpen, Filter, Score, etc.)
3. Register in `pkg/scheduler/plugins/factory.go`
4. Add configuration support in `pkg/scheduler/conf/`
5. Add tests and update documentation

### Adding New Controller
1. Create controller in `pkg/controllers/`
2. Implement controller interface from `pkg/controllers/framework/`
3. Define state machine in `state/` subdirectory
4. Register in controller manager
5. Add E2E tests

### Modifying APIs
1. Update API definitions in `volcano.sh/apis` dependency
2. Run `make generate-code` to update generated clients
3. Run `make manifests` to update CRDs
4. Update controllers and webhooks as needed

## Testing Guidelines

- **Unit Tests**: Located alongside source files (`*_test.go`)
- **E2E Tests**: Organized by feature in `test/e2e/`
- **Test Utilities**: Common helpers in `test/e2e/util/`
- **Mock Generation**: Use `github.com/golang/mock` for interface mocking

## Important Notes

- Volcano uses Kubernetes 1.23+ features and requires CRD support
- The project follows standard Kubernetes controller patterns
- Scheduling decisions are made on PodGroups, not individual Pods
- Plugin development requires understanding the session lifecycle
- Controllers use informers and work queues for scalability
- The webhook system provides both validation and mutation
- APIs are defined in external `volcano.sh/apis` dependency
- Version information is injected at build time via LD_FLAGS
- Plugin support can be enabled with `SUPPORT_PLUGINS=yes` (requires CGO)