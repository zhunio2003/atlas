# Contributing to Atlas

This is a solo project. This file exists to document conventions and keep things consistent — mostly for future-me at 2am.

---

## Prerequisites

- Go 1.23+
- `protoc` + `protoc-gen-go` + `protoc-gen-go-grpc` (for proto generation)
- `golangci-lint` v1.57+
- `make`

## Getting started

```bash
git clone https://github.com/zhunio2003/atlas.git
cd atlas
go mod download
make build
```

## Running tests

Always with `-race`. No exceptions.

```bash
make test                        # all tests
make test-raft                   # only internal/raft/...
make test-fuzz                   # fuzzing (runs for 30s by default)
```

Or directly:

```bash
go test -race -count=1 ./...
go test -race -count=1 ./internal/raft/...
go test -fuzz=FuzzRPCMessage -fuzztime=30s ./chaos/fuzzing/...
```

## Linting

```bash
make lint
# or
golangci-lint run ./...
```

Lint must pass before any PR merges. Config lives in `.golangci.yml`.

## Commit conventions

This project uses [Conventional Commits](https://www.conventionalcommits.org/).

```
feat: add leader election timeout randomization
fix: prevent double vote in same term
test: add chaos scenario for network partition
chore: update golangci-lint to v1.58
docs: add ADR-0003 for storage engine decision
refactor: extract election timer into separate goroutine
```

Breaking changes get a `!` after the type: `feat!: change RPC wire format`.

## Pull requests

Every change goes through a PR, even solo. Rules:

- CI must pass (tests + lint + build)
- Self-review before merging — at least read the diff once
- One logical change per PR; don't bundle unrelated things
- PR title follows the same Conventional Commits format

## Architecture decisions

Any significant decision (storage engine, consistency model, transport layer, etc.) gets an ADR before writing code.

ADRs live in `docs/architecture/adr-XXXX-title.md`. Use the template in that folder. The decision is documented first, the implementation follows.

## Repository structure

See [`ATLAS_project_brief.md`](./ATLAS_project_brief.md) for the project vision and [`docs/design.md`](./docs/design.md) for technical depth (invariants, failure model, consistency model, acceptance criteria per phase).

---

*Atlas · Zhunio · 2026*