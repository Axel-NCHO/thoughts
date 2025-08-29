# Proposal: Recovery-Oriented Programming 
_(or Zero-Fail Programming)_

## 1. Philosophy

Modern programming languages are largely designed around the happy path:  
- errors are treated as exceptions, interrupts that derail the main control flow.  
- The programmer is expected to explicitly detect, propagate, or catch them.

But in real-world systems like distributed networks, hardware, even biology, failure is the norm, not the exception.

The human brain, for instance, never expects every neuron to fire correctly. 
 It builds redundancy, parallel pathways, retries, and degradations into its wiring. 
 This ensures robustness because intent is carried through despite local failures.  

This proposal imagines a programming language (or paradigm) where:

- Failures are expected and recovery is the default behavior.
- Programmers express the intent of an operation and the system is responsible for fighting to realize that intent.
- Redundancy, fallback, and graceful degradation are core language features.
- Crashes occur only if the intent is truly impossible.

We call this **Recovery-Oriented Programming**.

---

## 2. Core principles

1. Failures are normal. Every failure-prone primitive is marked as such.
2. Recovery is structural. Functions that use failure-prone primitives must declare this.
3. Plain returns. Functions return plain values, not wrappers like `Result<T, E>`.
4. Recovery Pipelines. The system retries, falls back, and degrades automatically.
5. Concurrent Agents. Multiple recovery solutions may race, and the first win completes the intent.
6. Groups & Priorities. Recovery is organized into prioritized groups of agents.
7. Observability. All recovery actions are logged and queryable.
8. Learning. Successful recoveries are remembered in context, avoiding repeat effort.
9. Fatality is last resort. Only if no solution is possible does the system fatal.

---

## 3. Function semantics

### 3.1 Marking failure-prone functions

Primitives that may fail are marked with `!`:

```
def read(path: Path)! -> Bytes
def connect(url: String)! -> Connection
```

Any function that calls a `!` function must also be marked `!` unless it absorbs recovery.

```
def load_settings()! -> Settings:
    raw = read("/etc/settings")
    return parse(raw)
```

### 3.2 Absorbing Recovery

A function can handle recovery fully inside itself, removing `!`:

```
def safe_load_settings() -> Settings:
    return read("/etc/settings").recover_with(default_settings)
```

---

## 4. Recovery pipelines

### 4.1 Sequential & Concurrent recovery

Recovery is defined as a set of groups:

- Groups are prioritized (group 10 runs before group 20).
- Agents inside a group run concurrently and the first win cancels others.
- If all agents in a group fail, the next group executes.
- If all groups fail it's a fatal error.

```
policy network_default {
  group 10 "quick retries" {
    agent retry_1s { retry(times=5, delay=1s) }
  }

  group 20 "alternate routes" {
    agent dns_google { use_dns("8.8.8.8") }
    agent dns_cloudf { use_dns("1.1.1.1") }
  }

  group 30 "degrade" {
    agent offline_cache { serve_from_cache(max_stale=15m) }
  }

  fatal_if_all_fail
}
```

### 4.2 Remembering success

If a recovery solution succeeded once in a given context,
 it is remembered as the preferred solution for subsequent identical calls.

Example:

- First download("example.com") tried groups 10, 20 → dns_cloudf succeeded.
- Next download("example.com") uses dns_cloudf directly.

---

## 5. Programmer overrides

Programmers can inject or override recovery logic.

### 5.1 Local override

```
data = download("https://ex.com/a.pdf")
  .with_policy {
    group 5 "business fix" {
      agent warmup_edge { hit("/_health"); retry(2, delay=200ms) }
    }
  }
```

### 5.2 Function-level override

```
def get_article(id: Id)! uses network_default -> Article:
    return http.get(article_url(id))
      .override {
        group 15 "tenant-cache" {
          agent tenant_cache { serve_from_cache(max_stale=2m) }
        }
      }
```

---

## 6. Write safety

Writes are dangerous if retried concurrently.
 The language enforces write classes:

- read-only: always safe concurrently
- idempotent-write: requires operation ID for deduplication
- non-idempotent: system serializes or applies compensation (saga pattern)

```
def store(path: Path, data: Bytes)! uses storage_default -> Unit:
    write_file(path, data) as idempotent(op_id=hash(path, data))
```

---

## 7. Observability

- Recovery trace: Logs all groups/agents attempted, winners, timings.
- Provenance: Query runtime to know how a value was obtained.

```
trace = provenance(cfg)
// "served via mirror_b with dns=1.1.1.1"
```

---

## 8. Example: storage

```
policy storage_default {
  group 10 "retry local" { agent r { retry(3, backoff=200ms) } }

  group 20 "alternate fs" {
    agent raid { path_prefix("/mnt/raid") idempotent }
    agent ram  { path_prefix("ram://")    idempotent }
  }

  group 30 "degrade" { agent temp { path_prefix("/tmp/recovery") idempotent } }

  fatal_if_all_fail
}

def store(path: Path, data: Bytes)! uses storage_default -> Unit:
    write_file(path, data)

def read(path: Path)! uses storage_default -> Bytes:
    return read_file(path)
```

Usage:

```
store("/users/axel/settings", blob)
cfg = read("/users/axel/settings")
```

Even if the primary disk fails, reads/writes still succeed by redirecting transparently, and the data is always queryable at the same `path`, 
no matter where it's actually stored.  

---

## 9. Summary

This paradigm shifts programming from:

- “Handle errors explicitly everywhere” → Rust, Go  
   to
- “Trust the runtime to always carry the intent”

Key features:

- Failure is not exceptional.
- Recovery is structured, concurrent, prioritized.
- Functions either return a valid result or fatal.
- Recoveries are observable, learn from success, and customizable.

This model is inspired by biology and distributed systems, aiming for robustness first.
