# Architecture — Claude Autonomous Kit

> Visual map of how this repository actually works. For the broader vision (the
> always-on, 24/7 autonomous framework) see [`README.md`](README.md); for the live
> instruction contract see [`CLAUDE.md`](CLAUDE.md).

## What's actually in this repo

The README paints the full picture of a 24/7 autonomous operation (scheduled
sessions, GitHub-issue task intake, health daemons, auto-merge). **What is
checked into *this* repository today is the foundation that makes any of that
safe: a persistent memory system plus two session-lifecycle skills.** That
foundation is what this document describes precisely.

```mermaid
flowchart TD
    subgraph present["✅ In this repo (documented here)"]
        CM["CLAUDE.md / CLAUDE.example.md<br/>auto-loaded instruction contract"]
        SK[".claude/skills/<br/>onboard · handover"]
        MEM["memory/<br/>handovers · learnings · decisions"]
    end
    subgraph vision["🗺️ Described in README, not yet in this repo"]
        SCR["scripts/ (autonomous.sh, check_tasks.sh…)"]
        HK["hooks/ (session-start/end, scope-guard)"]
        UT["utils/ (action_registry, task_queue_manager)"]
        PL["plists/ (launchd schedules)"]
    end
    present -.->|"the memory layer the<br/>full framework builds on"| vision

    style present fill:#dfd,stroke:#1a1
    style vision fill:#eee,stroke:#999,stroke-dasharray:5
```

> ⚠️ **Discrepancy worth knowing:** the README's "File Reference" lists
> `scripts/`, `hooks/`, `utils/`, `templates/`, and `plists/`. Those directories
> are **not present** in this checkout — the repo currently ships only the memory
> system and skills below. Treat the README as the roadmap and this file as the
> as-built.

---

## 1. The core idea — memory survives, sessions don't

Each Claude Code session is **ephemeral**: it starts with no recollection of the
last one. This kit fixes that by writing the important parts of a session to
plain Markdown in `memory/`, so the next session can read its way back into
context. The contract is enforced by `CLAUDE.md` principle #1:
*"What isn't in `memory/` will be lost."*

```mermaid
flowchart LR
    S1["Session N"] -->|"/handover<br/>(write)"| M[("memory/")]
    M -->|"/onboard<br/>(read)"| S2["Session N+1"]
    S2 -->|"/handover"| M
    M -->|"/onboard"| S3["Session N+2"]

    style M fill:#2d6,stroke:#161,color:#000
```

---

## 2. Memory tiers — three kinds of remembering

```mermaid
flowchart TD
    M[("memory/")]
    M --> H["handovers/<br/><b>short-term</b> — per session"]
    M --> D["decisions/<br/><b>medium-term</b> — why we chose X"]
    M --> L["learnings/<br/><b>long-term</b> — reusable across projects"]

    H -->|"promote universal lessons"| L

    H --- HN["YYYY-MM-DD_summary.md:<br/>done · incomplete · decisions ·<br/>next-session start-here"]
    D --- DN["decision records with rationale"]
    L --- LN["patterns that outlive the project"]

    style H fill:#ffd,stroke:#aa0
    style D fill:#fdf,stroke:#a0a
    style L fill:#dff,stroke:#0aa
```

- **handovers/** — written at the end of every session by `/handover`. The
  working memory: what got done, what's still open, and the exact first action
  for next time. (`memory/handovers/EXAMPLE.md` is a sample; delete it once real
  ones exist.)
- **decisions/** — important choices with their reasoning, so future sessions
  don't re-litigate settled questions.
- **learnings/** — lessons general enough to apply to *other* projects.
  `/handover` explicitly checks whether anything from the session should be
  *promoted* here.

---

## 3. Session lifecycle — onboard → work → handover

```mermaid
sequenceDiagram
    participant U as Owner
    participant C as Claude Code
    participant CM as CLAUDE.md
    participant MEM as memory/

    Note over C,CM: CLAUDE.md auto-loads every session

    U->>C: /onboard
    C->>CM: read project overview + rules
    C->>MEM: read newest handover
    C->>MEM: read recent decisions
    C->>MEM: list learnings (read on demand)
    C-->>U: "Last time / still open / today's focus"

    rect rgb(235,245,255)
        Note over U,C: ... do the actual work ...
    end

    U->>C: /handover [optional note]
    C->>MEM: reconcile previous "incomplete" items
    C->>MEM: write YYYY-MM-DD_summary.md
    C->>MEM: propose promoting universal learnings
    C-->>U: handover path + next-session summary
```

### `/onboard` (`.claude/skills/onboard/SKILL.md`)
Read-only (`allowed-tools: Read, Glob`). Steps: read `CLAUDE.md` → read the newest
handover → read recent decisions → list learnings. Reports *last session / open
tasks / what to do now*. If there's no handover, it announces a first session.
Guidance baked in: **don't read everything**, only what's needed; ask rather than
guess.

### `/handover` (`.claude/skills/handover/SKILL.md`)
Read + write (`Read, Glob, Write, Edit`), accepts an optional free-text note
(`$ARGUMENTS`). Steps: reconcile last handover's open items → write a structured
note (`Completed / Incomplete / Decisions / Next session: start here / Notes`) →
check for learnings to promote → report the path. Guidance: **be concise, write
facts not hopes, never omit the "why."**

Both skills set `disable-model-invocation: true` — they run only when the owner
explicitly types `/onboard` or `/handover`, never autonomously.

---

## 4. The instruction contract (`CLAUDE.md`)

`CLAUDE.md` is loaded automatically at the start of every session and is the
top-priority instruction set. The committed version is in **Japanese**;
[`CLAUDE.example.md`](CLAUDE.example.md) is an English template to copy and
customize. Its three standing principles:

```mermaid
flowchart LR
    P1["1 · Record<br/>important things go in memory/<br/>or they're lost"]
    P2["2 · Confirm<br/>ask when unsure;<br/>don't guess and proceed"]
    P3["3 · Be concise<br/>direct answers over<br/>verbose explanation"]
    style P1 fill:#dfd
    style P2 fill:#ffd
    style P3 fill:#def
```

It also carries an owner profile (availability, technical level, goal) so the AI
can tailor its tone and autonomy to a non-engineer running things from a phone.

---

## 5. Directory structure (as built)

```mermaid
flowchart TD
    R["claude-autonomous-kit/"]
    R --> CMD["CLAUDE.md (active, JA)"]
    R --> CEX["CLAUDE.example.md (template, EN)"]
    R --> RM["README.md (full-framework vision)"]
    R --> LIC["LICENSE (MIT)"]
    R --> CL[".claude/skills/"]
    CL --> ON["onboard/SKILL.md"]
    CL --> HO["handover/SKILL.md"]
    R --> MEM["memory/"]
    MEM --> HV["handovers/ (+ EXAMPLE.md)"]
    MEM --> LE["learnings/ (.gitkeep)"]
    MEM --> DE["decisions/ (.gitkeep)"]

    style CMD fill:#ffd
    style CL fill:#efe
    style MEM fill:#dff
```

`.gitkeep` files keep the empty memory tiers under version control so the
structure exists on a fresh clone. Secrets (`.env`, `*.key`, `*.pem`,
`credentials.json`) are git-ignored by design.

---

## See also
- [`README.md`](README.md) — the full autonomous-operations vision and roadmap
- [`CLAUDE.md`](CLAUDE.md) — the live instruction contract (Japanese)
- [`CLAUDE.example.md`](CLAUDE.example.md) — English starter template
