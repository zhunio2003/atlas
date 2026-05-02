<div align="center">
    <h1>ATLAS</h1>
    <strong>Distributed key-value store with Raft consensus, built from scratch.</strong>
    <img src="docs/img/atlas.png" alt="logo atlas" width="900">
    <br/><br/>
     <!--<img src="https://img.shields.io/github/actions/workflow/status/zhunio2003/atlas/ci.yml?branch=main" alt="CI"> -->
     <img src="https://img.shields.io/github/go-mod/go-version/zhunio2003/atlas" alt="Go version"> 
     <img src="https://img.shields.io/github/license/zhunio2003/atlas" alt="License">
     <img src="https://img.shields.io/badge/status-work%20in%20progress-orange" alt="Status"> 
    <p align="center">
      <a href="#what-is-atlas">What</a> ·
      <a href="#why">Why</a> ·
      <a href="#quickstart">Quickstart</a> ·
      <a href="#architecture">Architecture</a> ·
      <a href="#roadmap">Roadmap</a> ·
      <a href="#docs">Docs</a>
    </p>
</div>

<h1 align="center">Atlas</h1>

<p align="center">
</p>

<p align="center">
</p>


---

<!--
  HERO / DEMO
  -----------
  Va aquí UN solo GIF/video corto (10–20s) que muestre el demo más fuerte que tengas
  en el momento. Empezar con el de leader election (Fase 1). Luego reemplazar por el
  de chaos + persistencia cuando exista.

  Si todavía no hay demo: poner una sola frase punzante en lugar del GIF.
-->

> **Status:** _Work in progress · Phase 1 (Leader Election)._  
> Atlas is an engineering portfolio project. Public progress, not a final release.

<!-- ![Atlas demo](docs/diagrams/leader-election.gif) -->

---

## What is Atlas <a id="what-is-atlas"></a>

<!--
  Una sola frase que diga qué es. Después un párrafo corto que diga qué produce.
  No vender todavía — describir.
-->

Atlas is a distributed key-value store that implements the **Raft** consensus protocol from scratch in Go. Write a key on node 1, read it from node 3 — even if node 2 crashed, even under network partition, even if the leader died mid-write. Strong consistency, real fault tolerance, no third-party consensus libraries.

**What Atlas ships:**

- `atlas-server` — a single binary that runs one node of the cluster
- `atlas` — CLI client (`atlas put foo bar`, `atlas get foo`)
- HTTP + gRPC APIs for external applications
- A Go client library (`pkg/client`)
- Observable consensus logs (Prometheus metrics, structured logging)
- A chaos test suite that proves the system doesn't lie under failure

---

## What Atlas is **not** <a id="non-goals"></a>

<!--
  Tabla del brief. Decir lo que NO es es tan importante como decir lo que es.
-->

| Not this                               | Why it matters to say so                          |
| -------------------------------------- | ------------------------------------------------- |
| A wrapper around etcd / Consul / hashicorp/raft | Using a Raft library defeats the point      |
| A relational database                  | No SQL, no schemas, no joins                      |
| A blob / file storage system           | Key-value only, small values                      |
| A Redis replacement                    | Redis is in-memory and single-node by design      |
| Production-ready                       | This is a learning + portfolio project            |

---

## Why <a id="why"></a>

<!--
  Sección corta. Por qué existe Atlas, por qué Raft, por qué desde cero.
  Honestidad > marketing.
-->

Raft powers **etcd** (the brain of Kubernetes), **CockroachDB**, and **TiKV**. Implementing it from scratch — not wrapping it — is one of the few things in distributed systems you cannot fake: either the cluster converges or it doesn't.

This repository is the working record of that implementation: code, tests, design notes, and the chaos suite that keeps it honest.

---

## Quickstart <a id="quickstart"></a>

<!--
  Cuando exista el binario, este bloque es lo primero que la gente prueba.
  Mientras no exista, dejarlo con `# coming in Phase 1` para que sea obvio.
-->

### Run a local 3-node cluster

```bash
# Clone
git clone https://github.com/zhunio2003/atlas.git
cd atlas

# Build
make build

# Bring up a 3-node cluster locally
docker compose -f deployments/docker-compose.yml up
```

### Use the CLI

```bash
# Put a value
atlas put greeting "hola mundo"

# Read it back from any node
atlas --node localhost:7002 get greeting
# → "hola mundo"

# Watch what happens when you kill the leader
./chaos/scenarios/kill_leader.sh
```

### Use the Go client

```go
import "github.com/zhunio2003/atlas/pkg/client"

c, _ := client.New(client.Options{
    Endpoints: []string{"localhost:7001", "localhost:7002", "localhost:7003"},
})
defer c.Close()

_ = c.Put(ctx, "greeting", []byte("hola"))
val, _ := c.Get(ctx, "greeting")
```

> See [`examples/basic-client`](examples/basic-client) for a full runnable example.

---

## Architecture <a id="architecture"></a>

<!--
  Diagrama simple ASCII al inicio (siempre se ve bien en GitHub).
  Después un diagrama real (Mermaid o imagen) que muestre los 3 nodos,
  el flujo de un Put, y el commit en el log.

  Mantener esta sección CORTA — el detalle real va en docs/design.md.
-->

```
                ┌──────────┐         ┌──────────┐         ┌──────────┐
   client ───►  │  node 1  │ ◄─────► │  node 2  │ ◄─────► │  node 3  │
                │ (leader) │         │ follower │         │ follower │
                └──────────┘         └──────────┘         └──────────┘
                     │                    │                    │
                     ▼                    ▼                    ▼
                  ┌─────┐              ┌─────┐              ┌─────┐
                  │ WAL │              │ WAL │              │ WAL │
                  └─────┘              └─────┘              └─────┘
                     │                    │                    │
                     └────── KV state machine (replicated) ────┘
```

**Layers, top to bottom:**

1. **Public API** — gRPC + HTTP, talks to clients, redirects to leader
2. **KV state machine** — applies committed log entries (`Put`, `Get`, `Delete`)
3. **Raft core** — election, replication, snapshots, log compaction
4. **Storage** — WAL + snapshot store, survives restarts
5. **Transport** — gRPC between nodes (swappable for in-memory transport in tests)

> Full design rationale: [`docs/design.md`](docs/design.md).  
> Architecture decisions and trade-offs: [`docs/architecture/`](docs/architecture/).

---

## Project structure <a id="layout"></a>

<!--
  Bloque colapsado. Solo el árbol top-level visible por defecto;
  los detalles se expanden con <details>. Esto evita que el README se infle.
-->

<details>
<summary><strong>Repository layout</strong> (click to expand)</summary>

```
atlas/
├── api/
│   └── proto/
│       ├── raft/v1/raft.proto
│       └── kv/v1/kv.proto
│
├── cmd/
│   ├── atlas-server/main.go
│   └── atlas-cli/main.go
│
├── internal/
│   ├── raft/
│   │   ├── node.go
│   │   ├── state.go
│   │   ├── election/
│   │   ├── replication/
│   │   ├── snapshot/
│   │   ├── log/
│   │   └── testutil/
│   ├── storage/
│   │   ├── wal/
│   │   ├── snapshot/
│   │   └── engine.go
│   ├── kv/
│   ├── transport/
│   │   ├── grpc/
│   │   └── transport.go
│   ├── server/
│   │   ├── http.go
│   │   ├── grpc.go
│   │   ├── auth.go
│   │   └── middleware.go
│   ├── config/
│   ├── observability/
│   │   ├── metrics/
│   │   ├── logger/
│   │   └── tracing/
│   └── errors/
│
├── pkg/
│   └── client/
│
├── chaos/
│   ├── scenarios/
│   └── fuzzing/
│
├── bench/
├── deployments/
│   ├── docker/
│   ├── docker-compose.yml
│   └── k8s/
├── scripts/
├── examples/
│   └── basic-client/
│
├── docs/
│   ├── design.md
│   ├── protocol.md
│   ├── architecture/
│   └── diagrams/
│
├── .github/workflows/
│   ├── ci.yml
│   └── release.yml
│
├── .golangci.yml
├── Makefile
├── go.mod
├── README.md
├── LICENSE
├── CONTRIBUTING.md
└── CHANGELOG.md
```
</details>

---

## Correctness <a id="correctness"></a>

<!--
  Esta sección es la diferenciadora. Es lo que separa "implementé Raft"
  de "implementé Raft y SÉ que funciona". Llenarla SOLO con lo que ya esté hecho.
-->

Raft is famously easy to misunderstand and hard to get right. Atlas treats correctness as a first-class concern.

**Safety properties verified:**

- [ ] Election Safety — at most one leader per term
- [ ] Leader Append-Only — leader never overwrites or deletes entries
- [ ] Log Matching — identical indices + terms ⇒ identical prefixes
- [ ] Leader Completeness — committed entries appear in all future leaders
- [ ] State Machine Safety — same index ⇒ same applied command, on every node

**Testing strategy:**

- Deterministic simulation (injected clock + injected network) — no `time.Sleep` in tests
- Property-based tests for log invariants
- Fuzzing on RPC message boundaries (`go test -fuzz`)
- Chaos scenarios: leader kill, network partition, slow links, packet loss
- Race detector (`-race`) on every CI run

> Full strategy: [`docs/design.md#testing`](docs/design.md#testing).

---

## Benchmarks <a id="benchmarks"></a>

<!--
  Vacío al inicio — está bien. Cuando Fase 2 termine, llenar con números reales.
  Mejor un número honesto que un número inflado.
-->

> Targets defined in [`docs/design.md#performance-targets`](docs/design.md#performance-targets).  
> Numbers will be published after Phase 2.

| Metric             | Target  | Measured |
| ------------------ | ------- | -------- |
| Write throughput   | TBD     | —        |
| Commit latency p50 | TBD     | —        |
| Commit latency p99 | TBD     | —        |
| Recovery time      | TBD     | —        |

---

## Docs <a id="docs"></a>

| Document                                            | Purpose                                          |
| --------------------------------------------------- | ------------------------------------------------ |
| [`ATLAS_project_brief.md`](ATLAS_project_brief.md)  | Vision document — what & why                     |
| [`docs/design.md`](docs/design.md)                  | Technical design — invariants, fault model, scope |
| [`docs/protocol.md`](docs/protocol.md)              | Wire protocol + Raft implementation notes        |
| [`docs/architecture/`](docs/architecture/)          | ADRs — one file per decision                     |
| [`CONTRIBUTING.md`](CONTRIBUTING.md)                | How to build, test, propose changes              |
| [`CHANGELOG.md`](CHANGELOG.md)                      | Version history (Keep a Changelog format)        |

---

## Development <a id="development"></a>

<!--
  Comandos del Makefile. Mantener sincronizado con el Makefile real.
-->

```bash
make build       # build binaries
make test        # go test -race -count=1 ./...
make lint        # golangci-lint
make proto       # regenerate gRPC stubs
make chaos       # run chaos scenarios against a local cluster
make bench       # run benchmarks
```

**Requirements:** Go 1.22+, `protoc`, `buf`, Docker (for local clusters).

---

## Acknowledgements <a id="acknowledgements"></a>

<!--
  Honestidad académica. Esto se ve bien y es correcto.
-->

- Diego Ongaro & John Ousterhout — _In Search of an Understandable Consensus Algorithm_ (Raft, 2014)
- MIT 6.5840 (formerly 6.824) — Distributed Systems labs, by Robert Morris
- Martin Kleppmann — _Designing Data-Intensive Applications_

---

## License <a id="license"></a>

<!-- Cuando elijas: MIT, Apache-2.0, etc. Hasta entonces dejar como TBD. -->

TBD — see [`LICENSE`](LICENSE).

---

<p align="center">
  <sub>Built by <a href="https://github.com/zhunio2003">zhunio2003</a> · 2026</sub>
</p>