# fleet-agent (source repo: `fleet-agent-early-version`)

A small Python package that ships two things: a **`BaseAgent`** class that every
"fleet" domain agent subclasses, and a **`fleet_math`** module of graph/geometry
helpers (`EmergenceDetector`, `HolonomyConsensus`, and a few functions). This is
the source for the `fleet-agent` package on PyPI. At least four other packages in
this org (`fishinglog-agent`, `activeledger-agent`, `reallog-agent`,
`capitaine-agent`) import and extend `BaseAgent`.

This README is an engineering guide. It tells you what the code *actually does*
(line by line, traced to source) and marks every capability so you know what you
can rely on:

- ✅ **real today** — does what it says, verified against source
- ⚠️ **real but conditional** — works, but with a caveat, boundary, or name that oversells it
- 🔮 **later phase** — described but not implemented

---

## ⚠️ Read this first: this repo is archived, but the package is published

The repository previously carried only this archive notice:

> **This repo is archived.** It was an early experiment that was never fully
> developed. Superseded by `tripartite-room` and `lighthouse-runtime`.
> *If there's code here, fork it. Run with it. The ideas were real — the
> implementations just didn't land.* Archived: 2026-05-13.

That notice and the package metadata disagree, and **both are true at the same
time**:

| What the archive notice says | What `pyproject.toml` says |
|---|---|
| Archived / superseded | `version = "0.2.2"`, `Development Status :: 4 - Beta` |
| "never fully developed" | Published to PyPI as `fleet-agent` |
| "the implementations just didn't land" | 4 sibling packages depend on `BaseAgent` |

This is the most important fact about the codebase. The implementation is real,
it imports, and people extend it — but it was declared finished-ish and then
declared abandoned, and the abandonment was not propagated to PyPI or to the
dependents. Treat this as **fork-and-own** territory, not a maintained upstream.
The archive notice's own suggestion — *fork it, run with it* — is the operating
model this README assumes.

---

## ⚠️ The name question (three names, one package)

| Where | Name |
|---|---|
| GitHub repository | `fleet-agent-early-version` |
| PyPI package (`pyproject.toml` `name`) | `fleet-agent` |
| `project.urls.Repository` in `pyproject.toml` | `github.com/SuperInstance/fleet-agent` (⚠️ not this repo — likely a dead/wrong link) |
| Import name | `fleet_agent` |

When you `pip install fleet-agent` you get this code; the "early-version" in the
repo name is a version qualifier, not a different project. If you follow the
Repository URL in the package metadata you will not land here.

---

## What's in the package

```
fleet_agent/
├── __init__.py     # re-exports BaseAgent + fleet_math symbols; __version__ = "0.2.0"
├── base.py         # BaseAgent, main_entry_point, setup_logging  (the part people extend)
└── fleet_math.py   # EmergenceDetector, HolonomyConsensus, Pythagorean48, graph helpers
example_agent.py    # a minimal working subclass
pyproject.toml      # name="fleet-agent", version="0.2.2", license text="MIT"
LICENSE             # ⚠️ AGPL-3.0 full text (see License section)
```

`__init__.py` re-exports `BaseAgent`, `main_entry_point`, `setup_logging`, and
the `fleet_math` symbols (`EmergenceDetector`, `HolonomyConsensus`,
`encode_pythagorean48`, `decode_pythagorean48`, `compute_h1_cohomology`,
`check_rigidity`, `optimal_neighbor_count`, `MAX_RIGID_NEIGHBORS`,
`BITS_PER_VECTOR`, `CONVERGENCE_CONSTANT`), so `from fleet_agent import ...`
works for all of them.

---

## Install & run

```bash
pip install fleet-agent          # published as 0.2.2
python example_agent.py --vessel oracle1 --domain oracle1_history --plato-url http://localhost:8847
```

⚠️ **Broken console script.** `pyproject.toml` declares
`[project.scripts] fleet-agent = "fleet_agent.base:FleetAgent.cli"`, but there is
**no `FleetAgent` class and no `cli` method anywhere in the source** (only
`BaseAgent`, which has no `cli`). After install, the `fleet-agent` command will
fail on import. Run agents as Python modules (`python example_agent.py …`) or via
`main_entry_point` instead. This is a packaging bug, not a usage error on your
part.

✅ **Zero runtime dependencies.** Despite the comment at the top of `base.py`
("No extra dependencies beyond `requests`"), the code uses only the Python
standard library (`urllib.request` / `urllib.error`). It does **not** import
`requests`, and `pyproject.toml` lists no runtime dependencies (only
`pytest` under an optional `dev` extra). Requires Python ≥ 3.10.

---

## Part 1 — `BaseAgent`: what a subclass inherits

`BaseAgent` (`base.py:24`) is a thin HTTP client around a **PLATO** tile server.
A "fleet agent" is a process that connects to PLATO, reads and writes **tiles**
(question/answer records) in a **room**, and does some domain-specific work in
an overridden `run()`.

### Construction and identity (`base.py:34`)

`BaseAgent(vessel, domain, plato_url="http://localhost:8847")` sets up, with no
I/O:

- ✅ `self.vessel`, `self.domain` — the two required identity strings.
- ✅ `self.plato_url` — trailing slash stripped.
- ✅ `self.agent_id = f"{vessel}@{domain}"` — the combined identifier.
- ✅ `self.started_at` — `datetime.now(timezone.utc).isoformat()` captured at construction.
- ✅ A stdlib logger `fleet_agent.{vessel}.{domain}` at `INFO`, writing to
  `stderr`, attached only if the logger has no handlers yet (so it won't double-
  attach under your own logging config).
- ✅ `self._connected = False` — connection state, initially false.

### Connection (`base.py:76`)

- ✅ `connect() -> bool` — issues `GET {plato_url}/room/{domain}` with a **5s
  timeout**. Sets `_connected = (status == 200)` and returns it. Returns `False`
  on `URLError` or any other exception (logged, not raised).
- ✅ `is_connected() -> bool` — returns `_connected`. Note: this reflects the
  *last* `connect()` result; nothing pings PLATO on each call.

### Tile operations (`base.py:120`, `base.py:169`)

- ✅ `read_tiles(room=None, limit=100, offset=0) -> list[dict]` — `GET
  {plato_url}/room/{room or domain}?limit=&offset=` with a **10s timeout**. On
  HTTP 200 returns `data["tiles"]` (a list); on any error (network, non-200,
  JSON decode, exception) returns `[]`. The empty-list-on-error contract means
  **you cannot distinguish "room is empty" from "PLATO is down"** from the return
  value alone — check the logs or `is_connected()`.

- ⚠️ `write_tile(question, answer, room=None, metadata=None) -> dict` — builds a
  tile `{domain, question, answer, agent: <self.vessel>, timestamp: <now>}`,
  then if `metadata` is supplied does **`tile.update(metadata)`**, then `POST
  {plato_url}/submit` with a **5s timeout**. On 200 returns the parsed JSON from
  PLATO (the code reads `result["status"]` and `result["room_tile_count"]`); on
  failure returns an `{"status": "error", ...}` dict.

  **⚠️ Gotcha — `metadata` can clobber the canonical tile fields.** Because the
  metadata dict is merged with `.update()` *after* `domain`/`question`/`answer`/
  `agent`/`timestamp` are set, a caller passing
  `metadata={"agent": "someone_else"}` rewrites the recorded author;
  `metadata={"domain": ...}` refiles the tile under another room;
  `metadata={"timestamp": ...}` backdates it. If you forward untrusted input
  into `metadata`, sanitize it first (strip those five keys). In trusted,
  first-party use this is invisible.

### Identity and CLI (`base.py:240`, `base.py:261`)

- ✅ `get_identity() -> dict` — returns `{vessel, domain, agent_id, started_at,
  plato_url, connected}`.
- ✅ `parse_args()` *(classmethod)* — argparse with `--vessel` (required),
  `--domain` (required), and `--plato-url` (default `PLATO_URL` env var, else
  `http://localhost:8847`).
- ✅ `from_args()` *(classmethod)* — `parse_args()` then `cls(vessel=…, domain=…,
  plato_url=…)`.

### The one thing you must implement: `run()` (`base.py:310`)

✅ `run()` is **abstract by convention**. The base implementation raises
`NotImplementedError(f"{ClassName}.run() must be implemented")`. Everything else
on `BaseAgent` is concrete and inherited; `run()` is the single seam where your
domain logic goes. A subclass that forgets to override `run()` will still
construct, connect, and parse CLI args fine — it only fails when something
actually calls `run()`.

### The standard entry point: `main_entry_point` (`base.py:338`, `base.py:322`)

✅ `main_entry_point(agent_class)` wires a subclass into a CLI:

```python
from fleet_agent import BaseAgent, main_entry_point

class MyAgent(BaseAgent):
    def run(self) -> None:
        tiles = self.read_tiles(limit=10)     # GET /room/<domain>
        # ...your domain logic...
        self.write_tile("status", "done", metadata={"role": "processor"})

if __name__ == "__main__":
    main_entry_point(MyAgent)
```

`main_entry_point` does: `setup_logging()` → `agent_class.from_args()` →
`agent.connect()` (exits 1 if it can't connect) → `agent.run()` (exits 1 on an
exception, 0 on `KeyboardInterrupt`). `setup_logging(level="INFO")` configures
stdlib `basicConfig` to `stderr`. `example_agent.py` is a runnable reference
subclass.

### The PLATO HTTP contract `BaseAgent` assumes

These are the endpoints the code actually calls (you need a server implementing
them, e.g. PLATO):

- ✅ `GET /room/{room}?limit=&offset=` → JSON with a `tiles` list (read).
- ✅ `GET /room/{room}` → 200 means "reachable" (connectivity check).
- ✅ `POST /submit` with a JSON tile body → JSON, ideally with `status` and
  `room_tile_count` (write).

`BaseAgent` is a client only; this repo does not contain a PLATO server.

---

## Part 2 — `fleet_math`: what it *actually* computes

`fleet_math.py` is described in its own header as "Fleet Mathematics — JC1-CT
Bridge insights." The names (`EmergenceDetector`, `HolonomyConsensus`) suggest a
distributed-coordination toolkit. **They are mostly small, correct, pure
functions with marketing-grade docstrings.** Here is what each one really does.

### Graph topology: the cyclomatic number ✅

✅ `compute_h1_cohomology(n_vertices, n_edges, n_components=1) -> int`
(`fleet_math.py:66`) returns `n_edges - n_vertices + n_components` when
`n_edges >= n_vertices`, else `0`. This is the **cyclomatic number** (the first
Betti number β₁) of a graph — i.e. the number of independent cycles. It is real,
standard graph theory, and the function computes it correctly given correct
inputs. (When `n_edges < n_vertices` it short-circuits to `0`.)

⚠️ `check_rigidity(n_vertices, n_edges) -> bool` (`fleet_math.py:78`) returns
`n_edges >= 2*n_vertices - 3` and is documented as "Laman's theorem." That edge
count is only the **global, necessary** half of Laman's theorem for 2D generic
rigidity — a full Laman check also requires *every* k-vertex subgraph to have ≤
2k−3 edges. So this returns `True` for graphs that have enough edges to be rigid
but aren't actually rigid. Treat it as a "passes the edge-count precondition"
check, not a rigidity proof.

✅ `optimal_neighbor_count() -> 12` (`fleet_math.py:83`) returns the constant
`MAX_RIGID_NEIGHBORS = 12`. It's a fixed value, not computed from anything.

### `EmergenceDetector` ✅ the math / ⚠️ the claims (`fleet_math.py:88`)

✅ `update(vertices, edges)` builds an adjacency list and runs a **breadth-first
search to count connected components** (`self.h0`), then sets `self.h1 =
compute_h1_cohomology(len(vertices), len(edges), self.h0)`. The BFS component
count is correct. So after `update`, `.h0` is the number of connected components
and `.h1` is the cyclomatic number — both real, both correctly computed.

- ✅ `.fully_formed` → `h1 == 0` (a graph with no independent cycles — a forest).
- ⚠️ `.emergence_detected` → `h1 > n_vertices // 2` (a **heuristic** threshold:
  "more independent cycles than half the vertices." Reasonable as a rule of
  thumb; not derived from any stated theory of emergence).
- ⚠️ `.confidence` → **always returns `1.0`** (literal constant, with the
  comment *"Math is certain, ML is probabilistic"*).

⚠️ The class docstring claims *"H1 cohomology: 100% accuracy, 2.7s BEFORE any
individual notices"* vs. a baseline *"cuda-emergence: 62% accuracy, 1.2s AFTER
visible."* **Nothing in this code measures accuracy, latency, or predicts
anything temporally.** `update` computes the current cyclomatic number of a graph
you hand it; that is all. The comparative numbers are not substantiated by
anything in this repository — read them as aspirations, not measurements.

### `HolonomyConsensus` ⚠️ (`fleet_math.py:144`)

⚠️ This is the biggest gap between name and behavior. The docstring claims it
*"replaces voting, CRDTs, BFT"*, with *"Latency: 38ms vs PBFT's 412ms"* and
*"Byzantine tolerance: any number vs 1/3."* **The class contains no networking,
no protocol, no voting, and no fault tolerance.** What it actually does:

- `add_tile(tile_id, holonomy=1.0)` stores a float per tile id, **defaulting to
  `1.0`**.
- `compute_cycle_holonomy(cycle)` multiplies the stored values for the tile ids
  in a caller-supplied cycle, **starting the product at `1.0`** and skipping any
  id not in `self.tiles`.
- `check_consensus(cycles)` returns `True` iff every cycle's product is within
  `tolerance` (default `1e-6`) of `1.0`.

Because both the default stored value and the empty product are `1.0`, a
freshly-constructed `HolonomyConsensus` reports **consensus on any cycles you
hand it, before any tile is added.** It is a tiny arithmetic predicate — "does
the product of these caller-supplied numbers round to 1" — not a consensus
algorithm. If you need actual distributed agreement, this is not it. If you want
a pure-function check that a set of per-edge scalars multiplies to ~1 around a
set of cycles, it does exactly that.

### Pythagorean48 encode/decode ⚠️ (the list has 44 entries, not 48)

⚠️ **Count mismatch.** The module is named and commented everywhere as "48"
(`PYTHAGOREAN_DIRECTIONS`, `encode_pythagorean48`, `decode_pythagorean48`,
`BITS_PER_VECTOR = log2(48)`), but **`len(PYTHAGOREAN_DIRECTIONS)` is actually
44** (verified by importing the module). The list contains 44 4-tuples
`(xn, xd, yn, yd)` — exact rational points on the unit circle (the cardinals
plus Pythagorean-triple directions like `(3,5,4,5)` → `(0.6, 0.8)`).

- ✅ `encode_pythagorean48(x, y) -> int` (`fleet_math.py:46`) — nearest-neighbour
  quantizer: iterates the **actual 44-element** list and returns the index
  `0..43` of the closest direction by squared Euclidean distance. Deterministic,
  and safe — it can never return an out-of-range index.
- ⚠️ `decode_pythagorean48(idx) -> (float, float)` (`fleet_math.py:60`) — returns
  `(xn/xd, yn/yd)` for `PYTHAGOREAN_DIRECTIONS[idx % 48]`. The modulo is **48**
  but the list has **44** entries, so `decode(44..47)` raises `IndexError`.
  `encode`'s output (always `0..43`) round-trips cleanly; an externally-supplied
  index of 44–47 does not.

⚠️ `BITS_PER_VECTOR = math.log2(48) ≈ 5.585` (`fleet_math.py:40`) describes a
hypothetical 48-direction set, **not the real 44-element list**
(`log2(44) ≈ 5.459`). It is also exported but unused by any function. The number
is arithmetically correct for 48; it just doesn't match what's in the list.

⚠️ `CONVERGENCE_CONSTANT = 1.692` (`fleet_math.py:43`) is **exported but never
used** by any function in the module. Its comment references "Ricci flow" and
"JC1 Law 103"; it's a dangling constant with no consumer here.

---

## Consolidated honesty ledger

Things to know before you depend on this, all verified against source:

| # | Item | Status |
|---|---|---|
| 1 | `BaseAgent` connect/read/write/identity/CLI | ✅ real today |
| 2 | `run()` is abstract; subclass must override or it raises | ✅ real today |
| 3 | Zero runtime deps (stdlib `urllib` only); `requests` is *not* used despite `base.py` comment | ✅ real today (comment is wrong) |
| 4 | `read_tiles` returns `[]` on *any* error — can't tell empty room from server down | ⚠️ real but conditional |
| 5 | `write_tile` `metadata` clobbers `agent`/`domain`/`question`/`answer`/`timestamp` via `.update()` | ⚠️ real but conditional |
| 6 | `compute_h1_cohomology` = cyclomatic number β₁; `EmergenceDetector.update` BFS components | ✅ real today |
| 7 | `PYTHAGOREAN_DIRECTIONS` has **44** entries but is named/commented/computed as 48; `decode` `% 48` IndexErrors on 44–47 | ⚠️ real but conditional |
| 8 | `check_rigidity` tests only the global edge count `E ≥ 2V−3` (necessary, not sufficient for Laman) | ⚠️ real but conditional |
| 9 | `EmergenceDetector.confidence` always returns `1.0`; "100% accuracy / 2.7s before" claims unsubstantiated | ⚠️ real but conditional |
| 10 | `HolonomyConsensus` is an arithmetic predicate, not a consensus protocol; defaults make it vacuously pass | ⚠️ real but conditional |
| 11 | `CONVERGENCE_CONSTANT` exported but unused | ⚠️ real but conditional |
| 12 | `[project.scripts] fleet-agent = fleet_agent.base:FleetAgent.cli` — `FleetAgent`/`cli` do not exist | ⚠️ broken (will error on run) |

### Version numbers disagree (⚠️)

The package reports different versions depending on where you look:

| Source | Version |
|---|---|
| `pyproject.toml`, `dist/`, built `PKG-INFO` | `0.2.2` |
| `fleet_agent/__init__.py` (`__version__`) | `0.2.0` |
| `STRUCTURE.md` | `0.1.0` |

`pip` will report `0.2.2`; code that reads `fleet_agent.__version__` will see
`0.2.0`. `STRUCTURE.md` is an older internal doc and is stale on this (and on the
dependency claim — it correctly says "standard library only").

---

## How the dependents use this

The four published sibling packages (`fishinglog-agent`, `activeledger-agent`,
`reallog-agent`, `capitaine-agent`) depend on `fleet-agent` and, per the
package's purpose, **subclass `BaseAgent`** and implement `run()`, inheriting
PLATO connection, tile I/O, identity, and the standard CLI/`main_entry_point`.
That is the extension contract this package exposes; this README does not audit
the dependents' internals.

---

## License (⚠️ conflicting)

- The `LICENSE` file in this repo is the full **GNU AGPL-3.0** text.
- `pyproject.toml` and the built `PKG-INFO` declare **MIT** (`license = {text =
  "MIT"}`, classifier `License :: OSI Approved :: MIT License`), and the archive
  notice in this repo's `AGENT.md`-adjacent docs also says MIT.

These are different licenses with different obligations (AGPL is copyleft with a
network-use clause; MIT is permissive). The file on disk is AGPL-3.0; the
metadata claims MIT. **Resolve this with the maintainer before redistributing** —
the two cannot both be the licence of this code.

---

## Quick reference — `BaseAgent` at a glance

| Member | Kind | Source | Notes |
|---|---|---|---|
| `vessel`, `domain`, `plato_url`, `agent_id`, `started_at` | attrs | `base.py:51` | set in `__init__`, no I/O |
| `connect()` | method | `base.py:76` | GET `/room/{domain}`, 5s timeout → `_connected` |
| `is_connected()` | method | `base.py:106` | returns last `connect()` result |
| `read_tiles(room, limit, offset)` | method | `base.py:120` | GET, 10s timeout → `list[dict]`, `[]` on error |
| `write_tile(question, answer, room, metadata)` | method | `base.py:169` | POST `/submit`, 5s timeout; ⚠️ metadata clobber |
| `get_identity()` | method | `base.py:240` | returns identity dict |
| `parse_args()` | classmethod | `base.py:261` | `--vessel --domain --plato-url` |
| `from_args()` | classmethod | `base.py:294` | construct from CLI args |
| `run()` | method | `base.py:310` | **abstract — override this** |
| `setup_logging(level)` | func | `base.py:322` | stdlib basicConfig → stderr |
| `main_entry_point(cls)` | func | `base.py:338` | wire a subclass to a CLI |

---

## Quick reference — `fleet_math` at a glance

| Symbol | Source | Actually computes | Status |
|---|---|---|---|
| `compute_h1_cohomology(V,E,C)` | `:66` | cyclomatic number β₁ = E−V+C | ✅ |
| `check_rigidity(V,E)` | `:78` | edge count `E ≥ 2V−3` only | ⚠️ necessary, not sufficient |
| `optimal_neighbor_count()` | `:83` | constant `12` | ✅ |
| `EmergenceDetector` | `:88` | β₁ via BFS; `confidence` always `1.0` | ✅ math / ⚠️ claims |
| `HolonomyConsensus` | `:144` | product of caller floats ≈ 1; no protocol | ⚠️ |
| `encode_pythagorean48(x,y)` | `:46` | nearest of **44** (not 48) unit directions | ⚠️ |
| `decode_pythagorean48(i)` | `:60` | inverse of encode; `% 48` IndexErrors on 44–47 | ⚠️ |
| `BITS_PER_VECTOR` | `:40` | `log2(48)` ≈ 5.585 (list has 44; unused) | ⚠️ |
| `MAX_RIGID_NEIGHBORS` | `:37` | `12` | ✅ |
| `CONVERGENCE_CONSTANT` | `:43` | `1.692`, exported, unused | ⚠️ |

---

*This README was written from the source in `fleet_agent/base.py` and
`fleet_agent/fleet_math.py`. Every method/constant description above is traced to
a file and line. Where the code's docstring makes a stronger claim than the code
supports, the docstring is marked ⚠️ and the actual behavior is described
instead.*
