# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NFS CSI (Container Storage Interface) Driver for Kubernetes. Enables dynamic provisioning and management of NFS volumes in Kubernetes clusters. Plugin name: `nfs.csi.k8s.io`.

## Build Commands

```bash
make                  # Build binary for current architecture (alias for `make nfs`)
make nfs              # Build NFS plugin binary to bin/${ARCH}/nfsplugin
make verify           # Run all verification (unit tests + hack/verify-all.sh)
make unit-test        # Run unit tests with coverage
make sanity-test      # Run CSI sanity tests (requires built binary)
make container        # Build multi-arch Docker images
make push             # Push images to registry
```

## Testing

```bash
# Unit tests with coverage
go test -covermode=count -coverprofile=profile.cov ./pkg/... -v

# Run a single test
go test -v ./pkg/nfs -run TestGetVolumeIDFromNfsVol

# CSI sanity tests
./test/sanity/run-test.sh

# E2E tests (requires Kubernetes cluster)
make e2e-bootstrap    # Deploy driver to cluster via Helm
make e2e-test         # Run E2E tests
make e2e-teardown     # Remove driver from cluster
```

## Local Development

```bash
# Start driver locally for testing with csc tool
./bin/nfsplugin --endpoint unix:///tmp/csi.sock --nodeid CSINode -v=5

# Test with csc (github.com/rexray/gocsi/csc)
csc identity plugin-info --endpoint "unix:///tmp/csi.sock"
```

## Verification Scripts

Located in `hack/`:
- `verify-all.sh` - Runs all checks
- `verify-gofmt.sh` / `update-gofmt.sh` - Go formatting
- `verify-govet.sh` - Go vet
- `verify-golint.sh` - Linting
- `verify-helm-chart.sh` - Helm chart validation

## Architecture

### CSI Service Implementation (3 gRPC Services)

**Entry Point:** `cmd/nfsplugin/main.go` - Parses flags, creates driver, starts gRPC server

**Core Package:** `pkg/nfs/`

| File | Purpose |
|------|---------|
| `nfs.go` | Driver initialization, configuration, capabilities |
| `server.go` | Non-blocking gRPC server implementation |
| `identityserver.go` | CSI Identity service (plugin info, probe) |
| `controllerserver.go` | Volume provisioning, deletion, snapshots, cloning |
| `nodeserver.go` | Mount/unmount volumes on nodes |
| `utils.go` | Volume ID parsing, mount options, locking |
| `tar.go` | Tar-based snapshot creation/restore |

### Volume ID Format

`server#share#subdir#uuid#onDelete`

Example: `192.168.1.100#/exports#pvc-abc123#uuid-456#delete`

### Key Abstractions

- **VolumeLocks** (`utils.go`): Mutex-based concurrent operation prevention per volume
- **Volume Stats Cache** (`nfs.go`): Timed cache for capacity reporting (default 10 min TTL)
- **Parameter Interpolation**: Supports `${pvc.metadata.name}`, `${pvc.metadata.namespace}`, `${pv.metadata.name}`

## Deployment

**Helm Charts:** `charts/latest/csi-driver-nfs/`

```bash
# Update chart package after changes
helm package charts/latest/csi-driver-nfs -d charts/latest/
```

**Manifests:** `deploy/` contains versioned kubectl manifests

## Key CLI Flags

`--endpoint` - CSI endpoint (default: unix://tmp/csi.sock)
`--nodeid` - Node identifier
`--drivername` - Driver name (default: nfs.csi.k8s.io)
`--mount-permissions` - Default mount permissions
`--default-ondelete-policy` - Volume deletion policy (delete/retain/archive)
