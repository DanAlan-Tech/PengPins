# Shucks' Root Run

**An interactive Linux certification trainer disguised as a cyberpunk infiltration game.**

A browser-based learning platform that teaches the Linux Professional Institute (LPI)
certification syllabus through a simulated shell. Every level is a real terminal on a
sandboxed box, every level hides a key, and every key is reachable only by actually
knowing the commands.

Think [OverTheWire](https://overthewire.org) or Hack The Box — but with a story, a cast,
and a syllabus that leads to a real certification.

![Neon Tundra splash art](assets/spashwork.jpg)

---

## Contents

- [What it is](#what-it-is)
- [Why I built it](#why-i-built-it)
- [What's built so far](#whats-built-so-far)
- [How it works](#how-it-works)
- [The engineering problem worth talking about](#the-engineering-problem-worth-talking-about)
- [Skills demonstrated](#skills-demonstrated)
- [Running it](#running-it)
- [Testing](#testing)
- [Project structure](#project-structure)
- [Roadmap](#roadmap)
- [Disclaimer](#disclaimer)

---

## What it is

A zero-dependency web application containing a **working shell emulator** — a simulated
Linux environment with a real filesystem model, process table, block devices, permissions
and a POSIX regular-expression engine. Learners type genuine Linux commands into a
genuine-feeling terminal and the environment responds the way a real box would.

Wrapped around that engine is a structured curriculum. Each level:

- maps to a **published LPI exam objective** (e.g. `103.3 Perform basic file management`)
- opens with a **briefing** in the voice of an in-world character
- tracks **objectives that tick as you perform them**, not as you finish
- hides a **key** (`PENG{...}`) inside the level's own data, reachable only once the
  technique is right
- ends with a **debrief** explaining what the commands did and why the exam asks about it
- awards a **reference card** to a persistent in-app Codex

The setting is the **Neon Tundra**: a frozen cyberpunk world where two factions fight over
network infrastructure. The player is a network infiltrator working toward the final
objective, Data Node 7. Each sector of the world corresponds to one LPI exam.

---

## Why I built it

I was studying for Linux certifications and ran into a problem that I suspect is common:
**a certification syllabus is a list, and lists are almost impossible to remember.**

Reading that `chmod` has a fourth permission digit does not make you able to use it.
Being handed a box where a script will not run until you work out *why* does.

So the design goal was not "gamify the syllabus" — it was to make the fiction and the
skill the same action:

> The antagonist has renamed a file to something awkward. To read it, you have to learn
> how to hand an awkward filename to a command. The sabotage **is** the exercise.

Three rules fell out of that and shaped every decision:

| Rule | What it means in practice |
|---|---|
| **Only real commands** | Nothing invented for convenience. Every command behaves as the real tool does, including the parts that are annoying. |
| **The key is in the material** | No level hands over a reward for finishing. The key already sits in the data; getting the technique right is what puts it in reach. |
| **Clear prompts, no free answers** | Hints are a ladder — the first press names the concept, the last gives the command. An automated test enforces that the first rung never contains a runnable command. |

---

## What's built so far

| Sector | Exam | Content | Status |
|---|---|---|---|
| **01 · The Thaw Point** | Linux Essentials 010-160 | 8 levels × 6 drills = **48 keys** | Complete |
| **02 · The Shell Fields** | LPIC-1 101-500 | 7 levels × 6 drills = **42 keys** | Complete |
| 03 · The Coolant Line | LPIC-1 102-500 | — | Planned |
| 04–07 | LPIC-2 / LPIC-3 | — | Planned |

**90 drills, 90 keys, 15 reference cards** across two sectors.

### Topics covered

**Sector 01** — navigation and hidden files · filenames that start with a dash · quoting
and shell metacharacters · `find` and directory traversal · `grep` · `sort`/`uniq`
pipelines · file permissions · a final **unguided capstone** that combines all of it.

**Sector 02** — stream redirection and file descriptors · text filters (`cut`, `tr`, `nl`,
`sed`) · file management and hard/symbolic links · processes, signals and scheduling
priority · regular expressions (BRE and ERE) · disks, filesystems, mounts and inodes ·
ownership, `umask` and the special permission bits.

### The capstone

Sector 01 ends with **Ghost run** — a field-clearance exam with no walkthrough. There is
nothing new in it; every drill recombines skills from earlier levels, and the briefings
state only the *end state* required. Hints deliberately refer the learner to their earned
reference cards rather than giving commands — a test verifies that **no hint rung in the
level contains a runnable command line.**

---

## How it works

**No build step. No dependencies. No backend.** Each sector is a single self-contained
HTML file that runs by opening it in a browser. Progress persists in `localStorage` under
a per-sector key.

The shell emulator inside implements:

| Subsystem | What it does |
|---|---|
| **Lexer / parser** | Quoting (single, double), backslash escapes, variable expansion, pipelines, redirection operators, trailing `&` |
| **Filesystem** | Permission modes, ownership, hard links sharing one inode, symbolic links that can dangle, hidden entries |
| **Redirection** | Modelled as **file descriptors**, not values — so `2>&1 > file` and `> file 2>&1` correctly differ |
| **Glob expansion** | Multi-segment patterns, with tracking of whether a path was reached literally or via wildcard |
| **Process table** | `ps` (three output formats), signals with correct trap semantics, niceness, job control, `nohup` |
| **Block devices** | `lsblk`, `df`, `du`, `mount`/`umount` that genuinely grafts a volume's files onto a mount point, inode accounting |
| **Regex engine** | A POSIX **BRE/ERE translator**, because JavaScript's regex engine is neither |
| **Permissions** | Full `chmod` including setuid/setgid/sticky and symbolic forms |

Terminal conveniences: command history, tab completion, piping, and a "core temperature"
mechanic that rises on failed commands and cools when a hint is used.

---

## The engineering problem worth talking about

Early on I wrote `grep` using JavaScript's built-in regular expressions. It worked. It was
also **teaching the wrong thing.**

In JavaScript, `a+b` means "one or more `a` followed by `b`". In POSIX Basic Regular
Expressions — what `grep` uses by default — the `+` is a **literal plus sign** until you
escape it. That distinction is explicitly tested on the LPIC-1 exam. A trainer that got it
backwards would actively teach a misconception.

That led to the principle the whole project now runs on:

> **Never assume the simulation is right. Compare it to the real tool.**

Every subsystem is **differentially tested** — the same input is run through the simulated
command and through the actual binary on a Linux host, and the outputs must match exactly.

| Checked against | Coverage |
|---|---|
| real `bash` | Stream routing, including the `2>&1 > f` vs `> f 2>&1` ordering case |
| real `grep` | **66 patterns** across BRE, ERE, fixed strings and POSIX character classes |
| real `sed` | **43 scripts** — `&`, backreferences, the numeric occurrence flag |
| real `chmod` / `umask` | **564 mode results**, special bits, symbolic and octal forms |
| real `ps` | All three header formats, byte for byte, including column widening |
| real `ls` / `stat` | Directory link counts, the `total` line, 4 KiB block arithmetic |
| real `du` / `df` / `lsblk` | Block totals, traversal order, human-readable scaling, header alignment |
| real coreutils | `cut`, `tr`, `nl`, `cp`, `mv`, `rm`, `mkdir`, `ln` |

**Every one of those harnesses found at least one genuine bug.** A sample of what only
showed up because the output was compared rather than eyeballed:

- `chmod` **preserves setuid/setgid on a directory** unless told otherwise. Clearing them
  numerically requires a *fifth* octal digit (`00755`) — `0755` is not enough. Sticky is
  not preserved. Files preserve nothing. I would never have guessed this; 15 test failures
  all turned out to be this single rule.
- `ls -l` prints a `total` line whose value is the sum of **allocated blocks**, not file
  sizes — and a directory's link count is `2 + (number of subdirectories)`.
- `ps` **widens its columns** to fit wide values, so a 341 MB RSS shifts the header.
- `sed`'s `&` in a replacement means "the whole match" — a subtle JavaScript
  `String.replace` footgun meant it was being silently dropped.
- `pgrep X` never matches itself, while `ps aux | grep X` always finds its own `grep`.

Alongside the differential suites, the whole curriculum is **played end to end in a
headless browser**: all 90 drills are solved with the intended commands, every objective
must tick, each key must be reachable from what the commands actually printed, and the
submission must be accepted. Additional checks cover HTML-escaping rules, hint leakage,
and horizontal overflow from 1440 px down to 360 px.

---

## Skills demonstrated

### Linux systems administration

Building a faithful simulation requires understanding the tools far more deeply than using
them does. To make a drill, I had to know not just what a command does but **how it fails,
what its exact error text is, and which edge cases the exam targets.**

- Shell streams, pipelines and file-descriptor semantics
- Filesystem internals: inodes, hard vs symbolic links, link counts, block allocation
- The full permission model including setuid, setgid, the sticky bit and `umask` arithmetic
- Process lifecycle, signal handling and trapping, scheduling priority, job control
- Storage: block devices, mount semantics, inode exhaustion as distinct from disk-full
- POSIX regular expressions and the BRE/ERE distinction
- The LPI Linux Essentials and LPIC-1 objective domains in depth

### Software engineering

- **Language implementation** — tokenizer, parser and evaluator for a shell grammar with
  quoting, escapes, pipelines, redirection and expansion
- **Systems modelling** — representing a filesystem, process table and block layer as
  data structures with correct invariants (e.g. two filenames sharing one inode)
- **State management** — per-sector progress, replay vs live runs, persistence
- Deliberate **zero-dependency architecture**: no framework, no build step, no backend —
  the whole thing runs from a file:// URL

### Test engineering

The part of this project I would most want to be asked about.

- **Differential testing** against reference implementations as the primary correctness
  strategy, rather than hand-written expectations that encode my own misunderstandings
- **Property-based assertions** where exact comparison was impossible — e.g. verifying
  that `ps` columns stay aligned with their header under any data width
- **Browser automation** (Playwright) driving the real UI for full end-to-end runs
- **Regression discipline** — the full suite runs after every change; a failing test was
  twice the thing that revealed the drill itself was wrong, not the engine
- Writing tests that **distinguish a bug in the code from a bug in the test**, which came
  up repeatedly and is a genuinely easy trap

### Frontend development

- Responsive layout verified down to 360 px, no framework involved
- A **design system** built on CSS custom properties with a consistent palette and type
  scale across both sectors
- Accessible interactive components — tabs with ARIA roles and keyboard support, live
  regions for terminal output
- A print stylesheet so learners can print the reference cards as physical study aids

### Instructional design

- Mapping every piece of content to a **published certification objective**
- Scaffolding difficulty across a sector and designing an **unguided capstone assessment**
- A **tiered hint system** that supports a stuck learner without removing the exercise
- Designing failure states that teach: error messages are written to tell the learner
  which concept they have actually hit

### Technical writing

- Developer documentation covering architecture, content format, design rules and the
  non-obvious behaviours that have caused bugs more than once
- In-app instructional copy — briefings, debriefs and reference cards

---

## Running it

No installation required.

```bash
git clone <repository-url>
cd <repository-folder>
```

Then open `index.html` in any modern browser — or serve the folder if you prefer:

```bash
python3 -m http.server 8000
# visit http://localhost:8000
```

Start at **Sector 01, Level 00**. Type `help` in the terminal for the command list,
`hint` if you get stuck, and `codex` to open your earned reference cards.

Progress is saved in your browser's local storage. Each sector uses its own key, so
clearing one does not affect the other.

---

## Testing

The test harnesses live in `tests/` and require Node.js and Playwright:

```bash
npm install playwright
npx playwright install chromium

node tests/test.js        # Sector 01 — all 48 drills end to end
node tests/s2test.js      # Sector 02 — all 42 drills end to end
node tests/recheck.js     # grep vs the system grep
node tests/permcheck.js   # chmod and umask vs the real tools
```

The differential harnesses invoke the host system's own binaries, so they need to run on
Linux (or WSL / a container) to compare against.

---

## Project structure

```
├── index.html          # Sector 01 — The Thaw Point (self-contained)
├── sector2.html        # Sector 02 — The Shell Fields (self-contained)
├── assets/             # Character art, concept art, splash art
├── docs/               # World lore, design brief, project compendium
├── study/              # Reference material and cheat sheets
└── tests/              # Test harnesses and recorded solutions
```

Each sector page carries its own copy of the terminal engine. This is a deliberate
trade-off: it keeps every page a single portable file with no shared runtime, at the cost
of applying shared fixes twice. Extracting the engine into a shared module is the planned
refactor once the sector count makes duplication the larger cost.

---

## Roadmap

- **Sector 03 — The Coolant Line** (LPIC-1 102-500): package management, systemd services,
  users and groups, scheduled jobs, networking
- Extract the shared terminal engine into a common module
- Spaced-repetition recall drills across cleared sectors
- Sectors 04–07 covering the LPIC-2 and LPIC-3 tracks

---

## Disclaimer

This is an **independent learning project**. It is not affiliated with, endorsed by, or
sponsored by the Linux Professional Institute. Mission objectives reference publicly
listed LPI exam objectives for study purposes only.

Exam codes and version suffixes change periodically — verify current objectives at
[lpi.org](https://www.lpi.org) before relying on them.

All security concepts are taught in a sandboxed, simulated environment. The project
includes an explicit code of conduct for learners: only test systems you own or are
authorised to test.
