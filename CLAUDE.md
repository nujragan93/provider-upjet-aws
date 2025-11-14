# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Provider-upjet-aws is a Crossplane provider for AWS, built using the Upjet code generation framework. It bridges AWS Terraform provider resources (currently v6.13.0) into Kubernetes as custom resources, supporting 150+ AWS services.

## Common Commands

### Building

```bash
# Build monolith provider (all services)
make build

# Build specific subpackages (recommended for development)
make build SUBPACKAGES="config ec2 s3"

# Build all subpackages
make build.all SUBPACKAGES="config ec2 rds"

# Build and load locally for testing
make build.all publish SUBPACKAGES="config ec2" \
  BUILD_ARGS="--load" \
  XPKG_REG_ORGS_NO_PROMOTE="" \
  XPKG_REG_ORGS="index.docker.io/$DOCKERHUB_ORG" \
  BRANCH_NAME=main
```

### Code Generation

```bash
# Generate all code (CRDs, controllers, examples) - REQUIRED after config changes
make generate

# Pull latest Terraform docs
make pull-docs

# Generate Terraform provider schema
make config/schema.json

# Update submodules (common build scripts)
make submodules
```

### Testing

```bash
# Run unit tests
make test

# Run linter
make lint

# Run end-to-end tests (requires AWS credentials)
# Set UPTEST_EXAMPLE_LIST to comma-separated list of example manifests
export UPTEST_CLOUD_CREDENTIALS="DEFAULT='[default]
aws_access_key_id = XXX
aws_secret_access_key = XXX'"
export UPTEST_EXAMPLE_LIST="examples/ec2/v1beta1/instance.yaml"
make uptest

# Run family e2e tests (builds and deploys providers, then runs uptest)
make family-e2e
```

### Running Locally

```bash
# Run monolith provider out-of-cluster (requires kubeconfig)
make run

# Run specific subpackage provider locally
make run-subpackage SUBPACKAGES=amp
```

### Linting and Formatting

```bash
# Run golangci-lint
make lint

# Format code
make fmt
```

## Architecture

### Dual-Scope Model

The provider uses a **dual-scope architecture** with parallel hierarchies:

1. **Cluster-scoped resources** (`apis/cluster/`, `config/cluster/`, `internal/controller/cluster/`)
   - API Group: `aws.upbound.io/v1beta1`
   - Global/cluster-level resources: IAM, organizations, route53, cloudfront, budgets, etc.
   - Use `ProviderConfig` from `aws.upbound.io/v1beta1`

2. **Namespace-scoped resources** (`apis/namespaced/`, `config/namespaced/`, `internal/controller/namespaced/`)
   - API Group: `aws.m.upbound.io/v1beta1`
   - Region-specific resources: EC2, RDS, S3, DynamoDB, Lambda, etc.
   - Use `ProviderConfig` from `aws.m.upbound.io/v1beta1`

Both contain the same ~170 AWS service packages, organized as separate subdirectories (ec2, s3, rds, iam, etc.).

### Service Packages (Subpackages)

Services are organized as independent packages that can be built/deployed separately:

- **Monolith**: Single provider binary with all services (default)
- **Per-service packages**: Individual binaries (e.g., `provider-aws-ec2`, `provider-aws-s3`)
  - Each is a complete provider for one service family
  - Built from `cmd/provider/{service}/`
  - Example: `SUBPACKAGES="config ec2 rds"` builds family config + EC2 + RDS providers

### Code Generation Pipeline

```
Terraform Provider (aws) v6.13.0 schema
    ↓
cmd/generator/main.go
    ↓ (reads schema.json)
config/{cluster,namespaced}/{service}/config.go
    ↓ (applies customizations)
Upjet Pipeline (pipeline.Run)
    ↓ (generates code)
apis/{cluster,namespaced}/{service}/v1beta1/ (CRDs)
internal/controller/{cluster,namespaced}/{service}/ (Controllers)
examples-generated/{cluster,namespaced}/ (Example manifests)
    ↓
cmd/provider/{subpackage}/zz_main.go (runtime)
    ↓
Crossplane Runtime (reconciliation)
```

**Key Points:**
- All code in `apis/`, `internal/controller/`, and `examples-generated/` is generated
- Manual changes should go in `config/{cluster,namespaced}/{service}/config.go`
- Run `make generate` after any config changes

### Configuration System

The `config/` directory controls resource generation:

1. **Service-specific configs**: `config/{cluster,namespaced}/{service}/config.go`
   - Each has a `Configure(p *config.Provider)` function
   - Registers resource customizations: references, external names, late initializers, schema tweaks

2. **Global configs**:
   - `config/overrides.go`: Applied to all resources (region requirement, tags handling, references)
   - `config/groups.go`: Maps Terraform names to Kubernetes API group/kind
   - `config/externalname.go`: Defines Terraform ID → Kubernetes identifier mapping
   - `config/schema.json`: Terraform provider schema (generated from Terraform binary)

3. **Provider aggregation**:
   - `config/cluster/provider.go` and `config/namespaced/provider.go`
   - Use `provider.AddConfig()` pattern to collect service configurations

### Key Components

- **cmd/generator/**: Code generation tool (invoked by `make generate`)
- **cmd/provider/{subpackage}/**: Provider runtime entry points
- **internal/clients/**: AWS SDK client setup, authentication, credential caching
- **internal/controller/**: Generated controllers (one per resource)
- **internal/features/**: Feature flags for gated functionality
- **hack/**: Build scripts and utilities
- **cluster/test/**: E2E test setup scripts

## Adding a New Resource

Follow the Upjet guide: https://github.com/crossplane/upjet/blob/main/docs/adding-new-resource.md

**Quick steps:**

1. Determine if resource should be cluster-scoped or namespaced
2. Create/update `config/{cluster,namespaced}/{service}/config.go`:
   ```go
   func Configure(p *config.Provider) {
       p.AddResourceConfigurator("aws_new_resource", func(r *config.Resource) {
           r.ShortGroup = "servicename"
           // Add references, external name config, etc.
       })
   }
   ```
3. Run `make generate` to generate CRDs and controllers
4. Build and test: `make build SUBPACKAGES="config servicename"`
5. Create example in `examples/servicename/v1beta1/`
6. Run tests: `export UPTEST_EXAMPLE_LIST="examples/servicename/v1beta1/resource.yaml" && make uptest`

## Authentication

The provider supports multiple authentication mechanisms (configured via `ProviderConfig`):

- **Secret**: Long-term IAM credentials from Kubernetes Secret (least secure)
- **IRSA**: IAM Roles for Service Accounts (EKS only, recommended)
- **PodIdentity**: EKS Pod Identity (EKS 1.24+, recommended)
- **WebIdentity**: Assumed Web Identity with OIDC tokens

See `AUTHENTICATION.md` for detailed configuration examples.

**Important**: Each scope (cluster/namespaced) has its own `ProviderConfig` CRD. Don't mix them.

## Development Workflow

### Making Config Changes

```bash
# 1. Edit config files
vim config/cluster/ec2/config.go

# 2. Regenerate code
make generate

# 3. Build affected subpackage
make build SUBPACKAGES="ec2"

# 4. Test locally (requires k8s cluster with Crossplane)
make run-subpackage SUBPACKAGES=ec2
```

### Debugging Tips

1. **Controller logs**: Check provider pod logs in `upbound-system` namespace
2. **Resource events**: `kubectl describe <resource>` shows reconciliation errors
3. **Terraform logs**: Set `spec.providerConfigRef.name` with debug logging enabled
4. **Schema inspection**: Check `config/schema.json` for Terraform resource schema

### Testing Patterns

- **Unit tests**: Go tests in `internal/` (mostly for custom logic, not generated code)
- **E2E tests**: Use `make uptest` with example manifests
- **Local testing**: Use `make run` or `make run-subpackage` with a local cluster
- **Breaking changes**: Run `make crddiff` to detect CRD schema breaking changes

## Important Files

- `Makefile`: Build targets, configuration variables, and task orchestration
- `go.mod`: Go dependencies (Crossplane runtime, Upjet, AWS SDK)
- `config/schema.json`: Terraform provider schema (regenerated when provider version changes)
- `config/generated.lst`: List of currently generated resources
- `internal/clients/aws.go`: AWS client setup and credential handling
- `internal/clients/pc_resolver.go`: ProviderConfig resolution logic

## Region Handling

- Most resources require a `region` field (enforced by `RegionRequired()` in `config/overrides.go`)
- Global resources (IAM, Route53, CloudFront) still accept region but ignore it
- Default region for examples: `us-west-1`
- Global resources/groups are listed in `internal/clients/aws.go`

## Build System

Uses **build submodule** (Upbound's common Makefiles):
- `build/makelib/*.mk`: Shared build logic for Go, Docker, XPKG, etc.
- Update with: `make submodules`
- Override variables in main `Makefile` as needed

## CI/CD

- **GitHub Actions**: See `.github/workflows/`
- **Build tagging**: Uses `buildtagger` tool for build constraints (when `RUN_BUILDTAGGER=true`)
- **Linting**: golangci-lint with custom configuration
- **Dependency caching**: Go module and build cache are preserved between runs

## Crossplane Integration

- Resources implement Crossplane XRM (Crossplane Resource Model)
- Support composition and claim-based provisioning
- Installed via Crossplane Package Manager: `pkg.crossplane.io/v1/Provider`
- Runtime uses Crossplane controller-runtime for reconciliation
- Webhooks for validation/defaulting (when enabled)

## Common Pitfalls

1. **Forgetting to run `make generate`**: Always regenerate after config changes
2. **Mixing cluster/namespaced scopes**: Each has separate APIs and ProviderConfigs
3. **Editing generated code**: Changes will be overwritten; edit `config/` instead
4. **Missing region**: Most resources require region field (set in examples and manifests)
5. **SUBPACKAGES syntax**: Use space-separated for Makefiles, comma-separated for scripts
6. **Stale schema**: Run `make config/schema.json` after Terraform provider version updates

## Resources

- Upjet documentation: https://github.com/crossplane/upjet/tree/main/docs
- Crossplane docs: https://docs.crossplane.io
- Provider marketplace: https://marketplace.upbound.io/providers/upbound/provider-aws/latest
- Monitoring guide: https://github.com/crossplane/upjet/blob/main/docs/monitoring.md
