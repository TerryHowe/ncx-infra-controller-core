# AGENTS.md

This file provides guidance for AI coding agents working in the
`ncx-infra-controller-core` repository.

## Project Overview

**NCX Infra Controller (NICo)** is an API-based microservice written in Rust
that provides site-local, zero-trust, bare-metal lifecycle management with
DPU-enforced isolation. It automates the complexity of the bare-metal lifecycle
to fast-track building next-generation AI Cloud offerings.

> **Status:** Experimental/Preview. APIs, configurations, and features may
> change without notice between releases.

### Key Responsibilities

- Hardware inventory management and orchestration
- Redfish-based hardware management
- Hardware testing and firmware updates
- IPv4 address allocation and DNS services
- Power control (on/off/reset)
- Provisioning, wiping, and node-release orchestration
- Machine trust enforcement during tenant switching

## Repository Structure

```
ncx-infra-controller-core/
├── crates/              # All Rust crate implementations (62 crates)
│   ├── carbide-api/     # Main gRPC API service
│   ├── carbide-agent/   # Background orchestration agent
│   ├── carbide-admin-cli/ # Administrator CLI tool
│   ├── carbide-scout/   # Machine validation and burn-in
│   ├── api-model/       # Shared API data models
│   ├── api-db/          # Database abstraction layer
│   ├── dhcp*/           # DHCP service implementation
│   ├── dns*/            # DNS services
│   ├── pxe/             # PXE boot image delivery
│   └── ...              # Additional service and utility crates
├── book/                # mdBook documentation
├── deploy/              # Kubernetes deployment configs and Kustomization overlays
├── dev/                 # Local dev tools (Dockerfiles, test configs, certs)
├── helm/                # Helm chart for Kubernetes deployment
├── bluefield/           # BlueField DPU-specific components
├── pxe/                 # PXE boot artifact generation
├── lints/               # Custom Clippy lints (carbide-lints crate)
├── include/             # Shared Makefile fragments
├── .github/             # GitHub Actions workflows and templates
├── Cargo.toml           # Workspace dependency management
├── Makefile.toml        # Primary build/task automation
├── Makefile-build.toml  # Build-specific tasks
└── Makefile-package.toml # Packaging tasks
```

## Technology Stack

- **Language:** Rust (edition 2024, toolchain pinned in `rust-toolchain.toml`)
- **Async runtime:** Tokio
- **gRPC framework:** Tonic (with TLS via Rustls/aws_lc_rs)
- **HTTP framework:** Axum (pinned to 0.8.4 for Tonic compatibility)
- **Database:** SQLx (compile-time checked queries)
- **Observability:** OpenTelemetry, Tracing (structured logfmt logging)
- **Build tool:** `cargo-make` (TOML task runner)
- **API definitions:** Protocol Buffers (protobuf)

## Build, Test, and Lint Commands

All task automation uses `cargo-make`. Install it with:

```bash
cargo install cargo-make
```

### Building

```bash
# Standard debug build (all workspace crates)
cargo build

# Release build
cargo build --release

# Full CI build + test (mirrors what CI runs)
cargo make build-and-test-release-container-services

# Build the admin CLI locally
cargo make build-cli
```

### Testing

```bash
# Run all tests
cargo test

# Build prerequisites first, then test (recommended for integration tests)
cargo make correctly-execute-tests
```

### Linting and Formatting

```bash
# Run all pre-commit checks (what CI runs)
cargo make pre-commit-verify-workspace

# Individual checks:
cargo make clippy              # Clippy linter (warnings = errors)
cargo make carbide-lints       # Custom carbide lints (requires nightly setup)
cargo make check-format-flow   # Check rustfmt formatting
cargo make check-format-nightly # Check import grouping/sorting (requires nightly)
cargo make check-workspace-deps # Validate dependency declarations in Cargo.toml
cargo make check-licenses      # Validate no restricted licenses introduced
cargo make check-bans          # Check for banned dependencies

# Auto-fix formatting:
cargo fmt --all
cargo make format-nightly      # Also sort imports
```

> **Note:** The nightly toolchain is used only for `check-format-nightly` and
> `carbide-lints`. The stable toolchain pinned in `rust-toolchain.toml` is used
> for everything else.

## Coding Conventions

See [`STYLE_GUIDE.md`](STYLE_GUIDE.md) for detailed conventions. Key points:

### Core Principles

- Prefer simple, explicit code over clever or heavily abstracted code.
- Design APIs that are hard to misuse — leverage the compiler.
- Do not add abstractions "just in case"; wait until there is a real requirement.

### Lints and Warnings

- All Clippy lints are enabled; all warnings are treated as errors.
- Avoid `#[allow(...)]` unless strongly justified.
- Avoid `#[allow(dead_code)]`; use `#[cfg(test)]` or `#[cfg(feature = "...")]`
  instead, or delete the unused code.

### Logging

Use structured logfmt logging. Pass common fields as attributes, not as string
interpolation:

```rust
// Preferred
tracing::error!(%machine_id, error=%e, "process_machine failed");

// Avoid
tracing::error!("process_machine {machine_id} failed: {e}");
```

### gRPC API Design

- List APIs follow the pattern: `FindResourceNameIds` + `FindResourceNamesByIds`.
- Each configurable resource object must have: `id`, `config`, `status`,
  `metadata`, `version`.
- State-managed resources additionally need: `state`, `state_version`,
  `state_reason`, `state_sla`.

### Error Handling

- API handlers use `CarbideError`, not raw `tonic::Status`.
- Map user errors to 4xx responses and system errors to 5xx responses.

### Database

- Use transactions for grouped writes; avoid long-running work inside transactions.
- Read-only functions should accept `impl DbReader`.
- **`api-db`** — wrapper functions; **`api-model`** — data definitions.
- The custom lint `txn_held_across_await` will catch transaction misuse.

### Async Code

- Prefer synchronous code over async when no I/O or timer is involved.
- Prefer `std::sync::Mutex` over `tokio::sync::Mutex` when contention is brief.

### Background Tasks

- Always join background tasks (panics in detached tasks do not propagate).
- Use a single `JoinSet` for all background tasks.
- Use `CancellationToken` for cancellation instead of `oneshot::Sender`.
- Naming convention: `start`/`spawn` indicates a task is spawned in background;
  `run` indicates a function runs forever.

### Crate Features

- Avoid crate features unless necessary; CI only tests default features.

### Metrics

- Avoid high-cardinality labels (e.g., per-machine attributes).

## Commit Guidelines

All commits **must** be signed with the Developer Certificate of Origin (DCO):

```bash
git commit -s -m "Your commit message"
```

DCO compliance is enforced automatically; unsigned commits block merging.

## Pull Request Guidelines

- Write PR descriptions as if the audience has no context: explain the *why*.
- Reference related issues.
- Keep PRs focused on a single change.
- Do not land unused code unless the PR is too large to review otherwise.
- Ensure all CI checks pass before requesting review.

## CI / CD

The primary CI workflow (`.github/workflows/ci.yaml`) runs on pushes to `main`
and release branches. It performs:

- Build and test for x86_64 and aarch64
- Clippy, format, license, and ban checks
- Secret scanning, CodeQL analysis, and Trivy container scanning
- Helm chart validation

## Further Reading

- [`README.md`](README.md) — Project overview and getting started
- [`STYLE_GUIDE.md`](STYLE_GUIDE.md) — Detailed Rust coding conventions
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — Contribution workflow and DCO process
- [`book/src/README.md`](book/src/README.md) — Architecture and operational guides
