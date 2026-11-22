# AINative instance 1 — migration plan

Baseline: panjf2000/gnet v2.10.0 @ 58b59a11350707153f8c48369693ab9beba7b2fa (Apache-2.0), pinned in `reference/gnet-go`.

Unit order from `rust-anything analyze` (schema rust-anything/analyze@1, 38 units, 22 ready, 9594 LOC) + fan-in/priority:

| Unit | Scope | Go source | Status |
|------|-------|-----------|--------|
| U0 | Scaffold: workspace+lints+ledger+verification dirs | — | this commit |
| U1 | Ring buffer (inbound read buffer) + math.CeilToPowerOfTwo | `pkg/buffer/ring/` (522 LOC), `pkg/math/` (64), `pkg/pool/byteslice` → Vec | **first PR** |
| U2 | Elastic buffer (outbound) | `pkg/buffer/elastic/` + `linkedlist/` | next |
| U3 | byteslice pool / goroutine pool | `pkg/pool/*` | next |
| U4 | Load balancer | `load_balancer.go` (117) | later |
| U5 | EventHandler trait + Action + options | `gnet.go` API surface, `options.go` | later |
| U6 | netpoll/epoll + eventloop + engine | `pkg/netpoll/`, `eventloop_unix.go`, `engine_unix.go`, `connection_unix.go` | last, real sockets |

Windows-only files (`*_windows.go`) are skipped — target is Linux epoll path; note in ledger.

U1 chosen over `internal/gfd`/`pkg/bs` start_here picks because ring buffer is the behavioral core (fan_in 4) and fully testable without sockets.
