---
name: explain-architecture
description: "Display software architecture, execution workflows, and runtime concurrency through protagonist-centered Graphviz DOT diagrams (architecture and flowcharts) and Mermaid sequence diagrams focused on observable I/O, execution boundaries, and boundary data. Supports optional detail mode (channel capacities and boundary field tables). Honor any explicitly requested diagram tool or structural format (schema, class diagram, file map)."
---

# Explain Architecture

Make the reader immediately see **where to start**, **what the primary actor wants**, and **how the system fulfills that objective**. Choose nodes for their behavioral role, resource identity, or execution step—not source files, classes, or crates, which serve only as evidence links unless the user explicitly asks for a structural map.

This skill governs **display only**. Even in detail mode, never introduce code-review judgments, bug findings, verdicts, or proposed fixes. Inside a broader review task, this skill governs only its diagrams and supporting factual details.

## 1. Establish Story and Evidence

- **Derive prominence from purpose**: Identify the requested behavior, scope, **primary actor (protagonist)**, and **main secondary actor** from the system's goal and the user's question—never from code size, dependency count, or startup chronology.
- **Ground every element in code**: Trace entry points, I/O calls, task creation/dispatch, channels, and shared state. Keep **implemented behavior**, **documented contracts**, **mocks**, and **external/unverified behavior** clearly distinguishable.
- **Never invent machinery**: Do not fabricate network protocols, thread assignments, queue drop policies, rollback, or atomicity to make a diagram look complete. State whether the view reflects **current source**, a **shipped release**, or **observed runtime behavior** whenever they could differ.

## 2. Select Form, Level, and Mode

Diagram **Form**, abstraction **Level**, and detail **Mode** are three orthogonal choices.

### 2.1 Diagram Form & Default Engine

Honor any explicitly requested tool or format (e.g., pure Mermaid, Draw.io, PlantUML, ASCII, schema, or class diagram). Otherwise route by question type:

| Form | Default Engine | When to Use | Node / Lifeline Semantics |
| :--- | :--- | :--- | :--- |
| **Architecture** | **Graphviz DOT** (`rankdir=LR`) | Participants, responsibilities, boundaries, topology | Behavioral roles, processes, channels, storage |
| **Flowchart** | **Graphviz DOT** (`rankdir=TB`) | Execution paths of an API, algorithm, or operation (branches, retries, recovery) | Coherent actions, explicit decisions, outcomes |
| **Sequence** | **Mermaid** (`sequenceDiagram`) | Temporal interaction order, waits, callbacks, responses, handoffs | Logical actors or repeated roles (a lifeline does **not** imply a dedicated OS thread) |

Combine forms only when each explains a distinct aspect. Never infer temporal ordering from a static architecture connection alone.

### 2.2 Abstraction Levels & Scope Routing

- **Default scope routing**: For a broad "explain the architecture" prompt, present **High + Medium** by default and offer **Low** for concurrency-heavy subsystems, rather than dumping every subsystem at all three levels at once. Jump straight to **Medium + Low** (or a single level) when the question targets concurrency, queues, or one operation.
- **Split rather than cram**: Keep each diagram focused (~12–15 nodes max). Split dense levels by interaction or subsystem instead of merging everything into one crowded graph. Preserve naming and state which higher-level actor a zoomed-in diagram expands.

| Level | Core Question | Required Visibility & Grouping Rules |
| :--- | :--- | :--- |
| **High** *(Conceptual)* | Who are the main characters, and what do they accomplish together? | Domain concepts and central objective. One logical actor may span processes or machines. Flowchart nodes represent business stages and key outcomes. Omit low-level machinery unless directly asked. |
| **Medium** *(Observable I/O)* | What crosses a boundary to produce the behavior? | Concrete processes, machines, users, files, devices/drivers, and external endpoints. Show every real boundary in scope: network I/O (HTTP/RPC/WS/TCP/UDP/QUIC/raw sockets, **listener/destination ports**, and **listener bind scope** such as `127.0.0.1` loopback vs. `0.0.0.0`/LAN/WAN), DB/cache, IPC, file R/W, UI I/O, DLL/module loading, process spawning, and **project-relevant device/kernel boundaries** (virtual NICs like TUN/TAP/Wintun, kernel drivers `.sys`/`.ko`, packet filters like WFP/NDIS/eBPF/Netfilter, `DeviceIoControl`/`ioctl` on `\\.\*` or `/dev/*`, kernel-user ring buffers, and directly accessed hardware/virtual devices). Keep supporting/fallback I/O visible but subordinate. Do **not** hide multiple external calls inside one generic flowchart step. |
| **Low** *(Execution & Sync)* | Where does work run, and how do execution contexts communicate? | Threads, event loops, executors, and worker pools (`subgraph cluster_*`); distinguish blocking workers vs. dedicated threads. Expose task handoffs, waits, connection ownership, channels (producers, consumers, element types), and shared state (readers, writers, locks/atomics) plus verified callback/ordering/backpressure behavior. Retain scoped Medium external endpoints so work traces through to observable I/O. |

### 2.3 Default Mode vs. Detail Mode

Use **Default mode** unless the user explicitly asks for **Detail mode** or its specific additions. **Requesting a Low-level view alone does not enable Detail mode.**

- **Both modes** must show **multiplicity** whenever repeated actors, connections, or tasks exist (e.g., `Worker threads x2`, `Reader tasks x active connections`). Distinguish **fixed counts**, **dynamic counts**, and **configured maximums**—never present a source limit as an observed runtime count.
- **Detail mode adds only**:
  1. **Channel capacities** on the diagram with units or `unbounded` / `unknown` (capacity ≠ current occupancy).
  2. **Boundary data appendix tables** keyed to shown diagram edges (file, IPC, network, ABI, `ioctl`/device/ring-buffer), listing only behavior-relevant fields with readable high-level names/meanings and source links to exact definitions. Never dump field inventories onto diagram nodes.

## 3. Non-Negotiable Modeling Laws

1. **Aggregation & P2P Symmetry**: Aggregate repeated actors **only** when their count is incidental/variable **and** their roles and relationships are fully interchangeable. Keep opposite endpoints and distinct ownership contexts separate even when running identical binaries. For one P2P connection, draw the **initiator** and **receiver** and each side's **local client + game/app** separately—never imply a shared local client, connection, or state that does not exist.
2. **In-Process Modules, Kernel/Device & OS Boundaries**: A user-mode DLL, shared library, or plugin runs **inside the process that loads it** (`cluster_*`), regardless of which application distributes its file on disk. By contrast, **kernel drivers (`.sys`/`.ko`), virtual NICs (TUN/TAP/Wintun), packet hooks (WFP/NDIS/eBPF), and device nodes (`\\.\*`, `/dev/*`) sit across the User ↔ Kernel / OS-stack boundary**—never place them inside a user-mode process `cluster_*`, even when paired with an in-process helper DLL (e.g., `wintun.dll` in-process vs. Wintun adapter/driver in kernel). Keep project-installed, opened, or traffic-intercepting drivers/devices explicitly visible—separating **control plane** (`ioctl`/driver load/route & filter rules) from **data plane** (packet capture, ring-buffer/`tun` I/O) where both exist—while collapsing generic OS services (like WMI) and standard OS filesystem/physical-NIC drivers behind ordinary file/socket calls unless central or requested.
3. **Ports, Connections & Sessions**: Surface **listener/destination ports**, transport protocols (TCP vs. UDP), and **listener bind scopes** (`127.0.0.1` loopback vs. `0.0.0.0`/external, or a listener binding `:0` and advertising its dynamic port) when a process exposes or connects to distinct service endpoints. **Omit ordinary outbound sender source ports and never draw standalone client-socket nodes**—surface a sender's local bind **only** when architecturally significant (e.g., a symmetric UDP/STUN/P2P port that also receives peer traffic, or explicit NIC binding like `SO_BINDTODEVICE`/`IP_UNICAST_IF` to bypass a virtual NIC). Remember **endpoint/port ≠ connection**: state whether interactions multiplex over **one shared connection** or open **separate connections** (even to the same `host:port`, e.g., `1 WebSocket per room`), and show setup/closure when relevant. In architecture diagrams, draw a **single bidirectional connection as one `dir=both` edge** labeled with transport/destination-port + messages each way (`Req -> / <- Event`), splitting only to clarify distinct behavior while noting the shared connection.
4. **Concurrency & State Accuracy**: Never imply one OS thread per coroutine/async task, and never pin a migrating pool task to one fixed worker; mark unverified placement as `unknown`. Draw task-owned local state inside the task—never as cross-thread shared state. Show wait/reject/drop backpressure only when verified in code; never guess from queue names.
5. **Workflow Branch, Retry & Unknown-Commit Rigor**: Give each action node one coherent purpose. Put explicit conditions in diamond nodes and label outgoing branches. Route retry edges back to the **exact step repeated** with the implemented attempt or deadline limit. Distinguish **definite rejection** from an **unknown commit result** (timeout/disconnect after send) and show actual recovery; never invent rollback or atomicity. Mark async dispatch, cleanup, and compensation with their true execution context and trigger conditions (do not imply background tasks finish before the response or that cleanup runs only on success unless proven). Split complex subprocesses while preserving parent stage name, entry, exits, and return position.
6. **Progressive Boundary Data**: Label important edges with real message/type names and purpose. At Medium level, note when wire/file/ABI encoding differs from in-memory structs. Link key actors, producers, consumers, and boundary types to source files.

## 4. Protagonist Composition & Engine Rules

### 4.1 Compose Around the Protagonists

- **Anchor the protagonist first**: Place the primary actor at the natural reading origin (`rank 0`: far-left in `LR`, top in `TB`) with the strongest border weight, followed immediately by the main secondary actor along a straight primary axis. Do **not** rely on `"start here"` text labels to rescue poor layout.
- **Keep upstream helpers subordinate**: Never let a config file, launcher, or DLL loader push the protagonist out of `rank 0` merely because its edge points into the protagonist.
  - *Example*: For a game SDK exposing host features, **Game** is primary and **Host** is secondary along the horizontal request/result spine; DLL loading and launcher relaunch enter perpendicularly as supporting paths. Only a question specifically about launching promotes **Launcher** to protagonist.

### 4.2 Graphviz DOT Mechanics (Architecture & Flowcharts)

Combine **rank-driven in/out separation** with **shortest-path boundary clipping** so edges stay short, untangled, and well-spaced:

- **Omit compass ports (`:n/:s/:e/:w`) by default (Shortest-Path Clipping)**:
  - Before `dot` runs, final 2D node positions are unknown—hardcoding `:e -> :w` or `:s -> :n` causes **180° wrap-around hooks** (when a target lands above or behind the source) and **single-point pinch knots** (forcing 4+ outgoing splines onto one center pixel of `:e`).
  - Leave edges **unported** (`A -> B`) so `dot` clips each spline to the **nearest facing border** and spreads multiple outgoing edges evenly along the node perimeter. Never attach compass ports to hub nodes (degree ≥ 3), cross-cluster edges, or diagonal/backward edges.
- **Separate In vs. Out via Rank Progression & Perpendicular Docking (not port syntax)**:
  - Forward rank order (`rank(upstream) < rank(actor) < rank(downstream)`) naturally brings inputs into the upstream side (`left` in `LR`, `top` in `TB`) and sends outputs out the downstream side.
  - Put `group="main"` and `weight=10, penwidth=2.0` on the **true synchronous primary spine** (not secondary background timers) so the core path stays straight and short.
  - Dock **at most one** pure local helper/lock beside an actor using `{ rank=same; Helper; Actor }` (with `dir=both` for acquire/release or R/W, unported) for a 1-hop perpendicular link; stacking two `rank=same` helpers on one actor forces skip-over arcs. Place helpers that also call downstream services at `rank(Actor) + 1` along their forward path.
- **Keep DAG edges constrained so `dot` minimizes crossings & guard `rank 0`**:
  - Weight primary synchronous calls (`weight=6..10`) so shared sinks (e.g., Redis/DB) sit near their call barycenter (1 hop from heavy callers). Keep forward async/cross-cluster edges **constrained** (default `constraint=true`, unported) so `dot`'s crossing-minimizer (`mincross`) vertically aligns producers with their target clusters instead of crossing lines across the canvas.
  - Because `constraint=false` both skips rank assignment (snapping zero-in-degree targets to `rank 0`) **and** bypasses crossing minimization, use `constraint=false` **only** on **cycle-creating retry/callback back-edges** and **rank-0 upstream spawners** (`Launcher -> Game [constraint=false]`). **Never** write `rank=same` across different `subgraph cluster_*` scopes.
- **Control hub fan-out, label clearance & container breathing room**:
  - Keep edge labels concise (≤ 22 chars/line, max 2 short lines, space-padded `" label "`) with `ranksep=1.0`, `nodesep=0.50`, and `edge [arrowsize=0.8]`. Never cram `if` prose or multi-command scripts onto architecture edges—put details in Markdown notes or pair with a Mermaid `sequenceDiagram` when numbered step order (`1..N`) matters. If one hub radiates > 6 edges, split cohesive sub-stages or collapse request/reply pairs into `dir=both`.
  - **Never use `splines=ortho`** (misplaces edge labels); use `splines=true` or `splines=polyline`. Name visible containers `subgraph cluster_<id>` with `labeljust="l"; labelloc="t"; margin=18; fontsize=12`. Always set `bgcolor="#FAFAFC"` and a CJK-safe sans-serif `fontname`; use bold rounded boxes for protagonists, rounded boxes for roles/actions, `shape=parallelogram` for channels/queues, `shape=cylinder` for storage/shared state, and `shape=diamond` for decisions.

**Pattern A — Medium Architecture (`rankdir=LR`, unported shortest-path + rank-driven in/out separation)**:

```dot
digraph arch {
    rankdir=LR; splines=true; nodesep=0.50; ranksep=1.00; pad=0.35; bgcolor="#FAFAFC";
    fontname="Segoe UI, -apple-system, PingFang SC, Microsoft YaHei, sans-serif";
    node [fontname="Segoe UI, -apple-system, PingFang SC, Microsoft YaHei, sans-serif", fontsize=11, shape=box, style="rounded,filled", fillcolor="#FFFFFF", color="#64748B", fontcolor="#0F172A", margin="0.18,0.12"];
    edge [fontname="Segoe UI, -apple-system, PingFang SC, Microsoft YaHei, sans-serif", fontsize=9.5, color="#334155", fontcolor="#334155", arrowsize=0.8];

    Launcher [label="Launcher\n(process)", color="#94A3B8", fontcolor="#475569"];
    subgraph cluster_game {
        label="Game Process"; labeljust="l"; labelloc="t"; fontsize=12; margin=18;
        style="rounded,filled"; color="#3B82F6"; fillcolor="#EFF6FF";
        Game   [label="Game Client", group="main", penwidth=2.6, color="#1D4ED8", fillcolor="#DBEAFE"];
        SdkDll [label="sdk.dll\n(in-process)", group="main", penwidth=1.6, color="#2563EB"];
        Game -> SdkDll [label=" SDK call ", weight=10, penwidth=2.0];
    }
    Host  [label="Host App\n(local process)", group="main", penwidth=1.8, color="#334155", fillcolor="#F8FAFC"];
    Cache [label="Session Store", shape=cylinder, fillcolor="#F1F5F9", color="#475569"];

    Launcher -> Game [label=" spawns ", style=dashed, color="#64748B", constraint=false];
    SdkDll -> Host [dir=both, label=" Named Pipe (1 per game) \n FeatureReq -> / <- EventPush ", weight=10, penwidth=2.0, color="#1E293B"];
    { rank=same; Host; Cache }
    Host -> Cache [dir=both, label=" R/W session ", style=dashed];
}
```

**Pattern B — Workflow Flowchart (`rankdir=TB`, straight spine + perpendicular side exits + bounded retry)**:

```dot
digraph flow {
    rankdir=TB; splines=true; nodesep=0.55; ranksep=0.70; pad=0.35; bgcolor="#FAFAFC";
    fontname="Segoe UI, -apple-system, PingFang SC, Microsoft YaHei, sans-serif";
    node [fontname="Segoe UI, -apple-system, PingFang SC, Microsoft YaHei, sans-serif", fontsize=11, shape=box, style="rounded,filled", fillcolor="#FFFFFF", color="#64748B", fontcolor="#0F172A", margin="0.18,0.10"];
    edge [fontname="Segoe UI, -apple-system, PingFang SC, Microsoft YaHei, sans-serif", fontsize=9.5, color="#334155", fontcolor="#334155", arrowsize=0.8];

    Start    [label="Submit Request", group="main", penwidth=2.6, color="#1D4ED8", fillcolor="#DBEAFE"];
    Validate [label="Validate & Send", group="main", penwidth=1.8, fillcolor="#F8FAFC"];
    AckCheck [label="Commit Ack?", shape=diamond, group="main", fillcolor="#FFF7ED", color="#C2410C"];
    Success  [label="Return Confirmed", group="main", penwidth=2.0, color="#15803D", fillcolor="#DCFCE7"];
    Reject   [label="Return Definite Error", color="#DC2626", fillcolor="#FEF2F2"];
    Poll     [label="Query Commit Status", color="#D97706", fillcolor="#FFFBEB"];

    Start -> Validate [weight=10, penwidth=2.0];
    Validate -> AckCheck [weight=10, penwidth=2.0];
    AckCheck -> Success [label=" committed ", weight=10, penwidth=2.0];

    { rank=same; Validate; Reject }
    Validate -> Reject [label=" rejected "];
    { rank=same; AckCheck; Poll }
    AckCheck -> Poll [label=" timeout (unknown) "];
    Poll -> AckCheck [label=" retry <= 3 ", style=dashed, constraint=false];
}
```

### 4.3 Rendering, Output Location & Mermaid Sequence Rules

- **Compile DOT to SVG and omit DOT source from chat**: Save `.dot` and `.svg` files in the session artifact folder or an OS temp subfolder (never inside an untracked repository tree unless the user asks to save project docs). **Never paste raw `.dot` source code into the chat window unless the user explicitly asks for it.** When `dot` is installed (`dot -Tsvg in.dot -o out.svg`), compile every DOT graph to verify zero syntax errors and surface only the rendered `.svg` link in Markdown. If `dot` is unavailable, link the saved `.dot` file path and mention installing Graphviz (`scoop install graphviz` / `winget install Graphviz.Graphviz`) in one short note without dumping the DOT code inline.
- **Mermaid `sequenceDiagram` rules**: Render sequence diagrams inline in ```` ```mermaid ```` blocks. Declare participants in left-to-right interaction order starting with the protagonist. Distinguish sync calls (`->>`), async/queued messages (`-)`), responses (`-->>`), and activation/waits. **Never put raw `;` in sequence messages or notes** (Mermaid parses `;` as a statement break—use `,`, `and`, or `#59;`).
- **No visual-inspection loops**: Checking `dot -Tsvg` exit status validates syntax; do **not** open browsers, capture screenshots, or iterate on rendered pixels.

## 5. Deliver and Verify

1. **Deliver concisely**: State the scope and provenance in 1–2 sentences, present the selected diagrams, explain **only non-obvious consequences and trade-offs** (do not narrate arrows the diagram already shows), link key implementations in source, and disclose any material evidence gaps. Architecture explanation alone requires neither modifying nor running project code.
2. **Pre-finish semantic & syntax check**:
   - DOT compiles cleanly (or passes syntax check) with no `splines=ortho`, no hardcoded `:n/:s/:e/:w` ports on hubs/cross-cluster/backward edges, at most one `rank=same` helper per actor, no cross-cluster `rank=same`, and no unintended rank-0 orphan nodes; Mermaid sequences contain no raw `;`.
   - Protagonist sits at `rank 0` with bold visual weight; main spine (`group="main"`, `weight=10`) runs straight without detours.
   - Chosen Form, Level, and Mode match the prompt; channels use parallelograms, storage uses cylinders, and single bidirectional transports use one `dir=both` edge with explicit connection multiplicity.
   - User-mode DLLs sit inside loader processes while kernel drivers/virtual NICs sit outside across the User ↔ Kernel boundary (with control vs. data plane distinguished); listener/destination ports and bind scopes are identified (without cluttering ordinary senders with client source ports or socket nodes); P2P peers remain separate; coroutines are not drawn as OS threads; flowcharts distinguish definite rejection from unknown commit and route retries to the exact repeated step with real limits.
   - Detail-mode capacities and appendix field tables appear **if and only if** requested, with working source links.
