# Feature: signposts — Knowledge Flow Maps

**Date:** 2026-02-17 (last updated 2026-02-17)

**Author:** Joshua Levy

**Status:** Draft

## Overview

signposts is a YAML format for defining **knowledge flow maps** — structured graphs of
documentation organized into workflows, paths, steps, and routes, navigable via CLI with
progress tracking.

Playbooks, runbooks, and onboarding guides have implicit structure: phases depend on
other phases, each phase has a checklist of steps, and which phase you go to next
depends on your situation.
Today this structure lives in prose ("proceed only if coverage >= 80%") and is invisible
to tooling. signposts makes it machine-readable.

**The format:**

- **`signposts.yml`** — the reusable graph definition.
  Lives in a knowledge repo (e.g., a playbook repository).
  Defines workflows, paths, steps, routes, and linked documentation.
- **`progress.yml`** — per-project state tracking.
  Lives in the consuming project.
  Tracks which paths and steps are complete, in progress, or skipped.

**The CLI:** `autorust signposts` navigates the graph, shows relevant documentation, and
tracks progress. `autorust guide` is a convenience alias that reads `signposts.yml`.

**Design philosophy:** signposts defines structure and links to content — it does not
contain the content itself.
Documentation stays in markdown files.
The YAML is the map; the docs are the territory.

**Key concepts:**

- **Workflow** — a group of related paths (e.g., "Getting Started", "Reference"). Organizes
  both display and traversal.
- **Path** — an ordered sequence of steps through documentation.
  The primary unit of work.
- **Step** — a single piece of documentation to work through (a docspec).
- **Route** — a conditional hint for jumping to another path or step, within the current
  path or across paths.

## Goals

- **General-purpose format**: Not porting-specific.
  Works for any knowledge-heavy, multi-phase workflow: porting playbooks, onboarding
  guides, migration checklists, security assessments.
- **CLI-navigable**: Agents and humans traverse the graph via CLI, receiving relevant
  documentation at each step.
- **Progress tracking**: Persistent state of where you are in the flow, what’s complete,
  what’s next.
- **Loose workflow**: Routes suggest what comes next but don’t enforce gates.
  You can jump to any path at any time.
  The graph is guidance, not a state machine.
- **Agent-friendly**: All output is structured for agent consumption.
  YAML for state, markdown for content, clear CLI semantics.

## Non-Goals

- **Enforced gates**: signposts does not block you from starting a path whose
  prerequisites aren’t met.
  The `when` conditions on routes are descriptive guidance for the human or agent, not
  machine-enforced preconditions.
- **Runtime orchestration**: signposts is not a workflow engine.
  It doesn’t execute tasks, schedule jobs, or manage agent coordination.
- **Content authoring**: signposts links to documentation; it doesn’t generate or
  template it.
- **Auto-generated graphs**: The graph is human-authored.
  This is intentional — the structure represents expert knowledge about how to navigate
  a domain.

## Background

### Prior Art

No existing tool combines all of these properties:

1. YAML-authored graph with workflows, paths, and routes
2. Agent-navigable via CLI
3. Steps/checklists within each path
4. Conditional branching (routes with `when` descriptions)
5. Persistent progress tracking across sessions
6. Documentation content linked at each path and step

The closest analogs from established tools:

- **XState** (statecharts): guarded transitions between states — the route/condition
  model.
- **GitHub Actions**: two-level hierarchy (jobs containing steps) with `needs` for
  dependencies and `if` for conditions.
- **Taskfile.yml** (go-task): `status` checks for progress, `preconditions` as guards.
- **Jupyter Book `_toc.yml`**: file-reference pattern — structure in YAML, content in
  docs.

The 2025-2026 agent tooling landscape has spec-driven development tools (Kiro, Spec Kit,
BMAD), agent knowledge graphs (Graphiti, AGENTiGraph), and flat instruction formats
(AGENTS.md, SKILL.md).
None combine authored graph structure with workflow navigation, checklists, and progress
tracking.

## Design

### `signposts.yml` — Graph Definition

Lives at the root of a knowledge repository (e.g., a playbook repo).
Defines the complete graph: workflows containing paths, each with steps and routes.

```yaml
# signposts.yml — Knowledge flow map for porting Python to Rust
#
# Workflows group related paths. Paths are ordered sequences of steps.
# Routes hint at how to jump between paths based on conditions.

format: "SP/0.1"
name: Python to Rust Porting Playbook
start: start

workflows:
  - id: start
    title: Start
    description: Determine your porting scenario
    docs:
      - ref#start-guide
    routes:
      - to: getting-started/assess
        when: Starting a new port from scratch
      - to: core-workflow/sync
        when: Syncing an existing port with Python upstream changes
      - to: reference/test-coverage
        when: Just need test coverage guidance

  - id: getting-started
    title: Getting Started
    description: Initial assessment and preparation
    paths:
      - id: assess
        title: Assess
        description: Understand scope, dependencies, and test coverage
        docs:
          - ./reference/python-to-rust-playbook.md
          - ./reference/python-to-rust-test-coverage-playbook.md
        steps:
          - ./reference/python-to-rust-playbook.md#dependency-risk-table
          - ./reference/python-to-rust-test-coverage-playbook.md
          - ref#go-no-go-criteria
        routes:
          - to: research
            when: Coverage sufficient, ready to evaluate Rust libraries
          - to: improve-coverage
            when: Test coverage below 80% on core modules

      - id: improve-coverage
        title: Improve Test Coverage
        description: Grow Python test coverage before porting begins
        docs:
          - ./reference/python-to-rust-test-coverage-playbook.md
          - ./guidelines/test-coverage-for-porting.md
        steps:
          - ./reference/python-to-rust-test-coverage-playbook.md#identifying-gaps
          - ./guidelines/test-coverage-for-porting.md#unit-tests
          - ./guidelines/test-coverage-for-porting.md#golden-tests
          - ./reference/python-to-rust-test-coverage-playbook.md#measuring-coverage
        routes:
          - to: assess
            when: Re-evaluate readiness after improving coverage

  - id: core-workflow
    title: Core Workflow
    description: The end-to-end porting process
    paths:
      - id: research
        title: Research
        description: Evaluate Rust library candidates with real inputs
        docs:
          - ./reference/python-to-rust-playbook.md
          - ./reference/python-to-rust-mapping-reference.md
        steps:
          - ./reference/python-to-rust-mapping-reference.md
          - ./reference/python-to-rust-playbook.md#library-evaluation
          - ./reference/python-to-rust-playbook.md#library-decisions
        routes:
          - to: plan
          - to: assess
            when: Risk too high, reconsider scope

      - id: plan
        title: Plan
        description: Architecture, module order, feature parity matrix
        docs:
          - ./reference/python-to-rust-playbook.md
        steps:
          - ./reference/python-to-rust-playbook.md#module-porting-order
          - ./reference/python-to-rust-playbook.md#feature-parity-matrix
          - ./reference/python-to-rust-playbook.md#effort-estimation
        routes:
          - to: setup

      - id: setup
        title: Set Up
        description: Cargo.toml, CI, test fixtures, Python submodule
        docs:
          - ./guidelines/rust-project-setup.md
          - ./reference/rust-cli-best-practices.md
        steps:
          - ./guidelines/rust-project-setup.md#cargo-init
          - ./reference/rust-cli-best-practices.md#ci-cd
          - ./reference/python-to-rust-porting-guide.md#shared-fixtures
          - ./guidelines/rust-project-setup.md#submodules
        routes:
          - to: port

      - id: port
        title: Port
        description: Port module by module, tests first, leaf to root
        docs:
          - ./reference/python-to-rust-porting-guide.md
          - ./guidelines/python-to-rust-porting-rules.md
        steps:
          - ./reference/python-to-rust-porting-guide.md#leaf-modules
          - ./reference/python-to-rust-porting-guide.md#integration-modules
          - ./guidelines/python-to-rust-cli-porting.md
        routes:
          - to: fix
          - to: research
            when: Library gap discovered during porting

      - id: fix
        title: Fix
        description: Cross-validate outputs, categorize diffs, workarounds
        docs:
          - ./reference/python-to-rust-playbook.md
          - ./reference/python-to-rust-porting-guide.md
        steps:
          - ./reference/python-to-rust-porting-guide.md#cross-validation
          - ./reference/python-to-rust-playbook.md#categorizing-differences
          - ./reference/python-to-rust-playbook.md#workaround-strategies
          - ./reference/python-to-rust-playbook.md#accepted-differences
        routes:
          - to: finalize
          - to: port
            when: Porting bugs found, need to revisit module implementation

      - id: finalize
        title: Finalize
        description: CLI parity, docs, release configuration
        docs:
          - ./reference/rust-cli-best-practices.md
          - ./reference/rust-code-review-checklist.md
        steps:
          - ./reference/rust-cli-best-practices.md#cli-parity
          - ./reference/rust-cli-best-practices.md#documentation
          - ./reference/rust-cli-best-practices.md#release-configuration
          - ./reference/rust-code-review-checklist.md
        routes:
          - to: sync

      - id: sync
        title: Sync
        description: Track Python upstream updates, manage divergences
        docs:
          - ./reference/port-checklist-update-template.md
        steps:
          - ./reference/port-checklist-update-template.md#check-upstream
          - ./reference/port-checklist-update-template.md#evaluate-changes
          - ./reference/port-checklist-update-template.md#port-updates
          - ./reference/port-checklist-update-template.md#cross-validate
        routes:
          - to: fix
            when: New differences found after syncing upstream changes

  - id: reference
    title: Reference
    description: Comprehensive reference documents
    paths:
      - id: mapping-reference
        title: Mapping Reference
        description: Python-to-Rust type/pattern/construct mapping
        docs:
          - ./reference/python-to-rust-mapping-reference.md

      - id: test-coverage
        title: Test Coverage
        description: Test coverage strategy and golden testing for porting
        docs:
          - ./reference/python-to-rust-test-coverage-playbook.md
          - ./guidelines/test-coverage-for-porting.md

      - id: cli-best-practices
        title: CLI Best Practices
        description: Rust CLI best practices and crate recommendations
        docs:
          - ./reference/rust-cli-best-practices.md

      - id: code-review
        title: Code Review
        description: Rust code review checklist
        docs:
          - ./reference/rust-code-review-checklist.md

  - id: guidelines
    title: Guidelines
    description: Compact rules suitable for agent context injection
    paths:
      - id: porting-rules
        title: Porting Rules
        description: Python-to-Rust porting rules and pitfalls
        docs:
          - ./guidelines/python-to-rust-porting-rules.md

      - id: cli-porting
        title: CLI Porting
        description: CLI-specific porting (argparse to clap, SIGPIPE, exit codes)
        docs:
          - ./guidelines/python-to-rust-cli-porting.md

      - id: rust-general
        title: General Rust Rules
        description: General Rust coding rules (Edition 2024+)
        docs:
          - ./guidelines/rust-general-rules.md

      - id: rust-cli-patterns
        title: Rust CLI Patterns
        description: Rust CLI application patterns (clap, error handling, config)
        docs:
          - ./guidelines/rust-cli-app-patterns.md

      - id: rust-project-setup
        title: Rust Project Setup
        description: Cargo.toml, CI/CD, lint config, release workflow
        docs:
          - ./guidelines/rust-project-setup.md

  - id: templates
    title: Templates
    description: Templates for case study documentation
    paths:
      - id: observations-template
        title: Observations Template
        description: Template for recording port observations (playbook feedback)
        docs:
          - ./reference/case-study-observations-template.md

      - id: triage-template
        title: Triage Template
        description: Template for triaging observations into playbook improvements
        docs:
          - ./reference/case-study-improvement-triage-template.md

  - id: meta
    title: Meta
    description: Maintaining and improving the playbook itself
    paths:
      - id: improving-playbook
        title: Improving the Playbook
        description: How to improve this playbook through your port experience
        docs:
          - ./reference/meta-improving-this-playbook.md

refs:
  - id: start-guide
    title: Getting Started with Your Port
    body: |
      Welcome to the Python-to-Rust porting playbook!

      **What brings you here?**
      - Starting a fresh port → Go to Assess (getting-started workflow)
      - Syncing an existing port with Python changes → Go to Sync (core-workflow)
      - Need test coverage help → Go to Test Coverage (reference)

      Use `autorust signposts where` to see your current position and next steps.

  - id: go-no-go-criteria
    title: Go/No-Go Decision Criteria
    body: |
      Before proceeding to the Research phase, confirm:
      - [ ] Test coverage >= 80% on core modules
      - [ ] All critical dependencies have Rust equivalents or viable alternatives
      - [ ] Team has basic Rust familiarity (completed rustlings or equivalent)
      - [ ] Timeline allows for 2-3x estimated effort (porting uncertainty buffer)
```

### Schema Reference

#### Top-level fields

| Field | Required | Description |
| --- | --- | --- |
| `format` | yes | Format version (currently `SP/0.1`). Checked for compatibility. |
| `name` | yes | Human-readable name for this knowledge flow map. |
| `start` | yes | Workflow ID of the entry point. Traversal begins at the first path in this workflow. |
| `workflows` | yes | Ordered list of workflow definitions. |
| `refs` | no | List of inline document definitions, referenced by `ref#<id>` docspecs. |

#### Document References: Docspecs and Locspecs

All document references in signposts use a two-part format:

```
<docspec>[#<locspec>]
```

A **docspec** identifies a document.
It is one of:

- **Relative file path**: `./reference/python-to-rust-playbook.md` — the `./` prefix
  identifies this as a file path, resolved relative to the directory containing the
  `signposts.yml` file

- **URL**: `https://docs.rs/comrak/latest/comrak/`

- **Local ref**: `ref#<id>` — references an inline document defined in the `refs`
  section of the same signposts.yml file.
  Useful for short content that doesn’t warrant its own file (decision criteria, quick
  checklists, brief notes).
  No YAML quoting needed.

- **Remote source** using the docspec format for references:

  ```
  github:owner/repo@ref//path
  ```

  This follows GitHub Actions (`@ref`), Nix flakes (`github:` prefix), and Terraform
  (`//path`) conventions, avoiding the slash-in-branch-name ambiguity of GitHub web
  URLs.

  In config, docspecs can decompose into `url` + `ref` + `paths` fields for readability:

  ```yaml
  # These are equivalent:
  # docspec: github:jlevy/rust-porting-playbook@main//reference/playbook.md
  - type: repo
    url: github.com/jlevy/rust-porting-playbook
    ref: main
    paths: [reference/playbook.md]
  ```

A **locspec** (optional) identifies a location within the document, delimited by `#`:

- By default, a GFM (GitHub Flavored Markdown) heading slug:
  `./reference/python-to-rust-playbook.md#phase-2-research`
- The slug is derived from the heading text using standard GFM rules (lowercase, spaces
  to hyphens, strip punctuation).

Future: explicit IDs via HTML comments (e.g., `<!-- id: my-anchor -->`) may be
supported, but for now locspecs are GFM heading slugs only.

**Examples:**

```yaml
# Local file path
- ./reference/python-to-rust-playbook.md

# Local file path with locspec
- ./reference/python-to-rust-playbook.md#dependency-risk-table

# Local ref (defined in refs section)
- ref#go-no-go-criteria

# URL
- https://docs.rs/comrak/latest/comrak/

# Remote GitHub source
- github:jlevy/rust-porting-playbook@main//reference/playbook.md

# Remote GitHub source with locspec
- github:jlevy/rust-porting-playbook@main//reference/playbook.md#phase-2-research
```

All YAML keys in signposts use `snake_case` convention.

#### Workflow fields

| Field | Required | Description |
| --- | --- | --- |
| `id` | yes | Globally unique identifier. Must match `[a-z][a-z0-9-]*`. |
| `title` | yes | Short display title. |
| `description` | no | One-line description shown in listings. |
| `docs` | no | List of docspecs (with optional locspecs). Documentation for the workflow itself (e.g., overview, decision criteria). |
| `routes` | no | List of routes to paths (in this or other workflows). Used for menu-style navigation or conditional workflow selection. |
| `paths` | yes | Ordered list of path definitions. |

Workflows organize both display and traversal.
The default traversal order within a workflow is the listed path order.

Workflows with `docs` and `routes` can serve as decision points or menus, guiding users/agents
to the appropriate workflow based on their situation.

#### Path fields

| Field | Required | Description |
| --- | --- | --- |
| `id` | yes | Globally unique identifier. Must match `[a-z][a-z0-9-]*`. |
| `title` | yes | Short display title. |
| `description` | no | One-line description shown in listings. |
| `docs` | yes | List of docspecs (with optional locspecs). |
| `steps` | no | Ordered list of steps (docspec strings). |
| `routes` | no | List of routes to other paths or steps. |

Paths without `routes` are terminal within their workflow (default traversal continues to
the next path in the workflow, or the workflow is complete).
Paths without `steps` are informational (reference material, no checklist).

#### Step fields

A step is simply a docspec (with optional locspec).
The `docspec#locspec` string serves as the step’s identity — no separate `id` field is
needed. The step’s display text comes from the doc’s frontmatter `description` or the
heading text when using a locspec.

Steps within a path must be unique (no duplicate docspec#locspec pairs).

#### Route fields

Routes can appear in workflows (workflow-level routes) or paths (path-level routes).

| Field | Required | Description |
| --- | --- | --- |
| `to` | yes | Target path. Format: `path-id` (within current workflow) or `workflow-id/path-id` (cross-workflow). |
| `at` | no | Target step (docspec#locspec) within the target path. If omitted, starts at the beginning of the path. |
| `when` | no | Human-readable condition describing when to take this route. If omitted, this is the default/unconditional next step. |

**Path-level routes** (routes within a path) can:
- Jump within the current path (same `to` as current path, different `at`)
- Jump to another path in the same workflow (just the path ID)
- Jump to a path in a different workflow (`workflow-id/path-id`)

**Workflow-level routes** (routes within a workflow) must use the `workflow-id/path-id`
format or reference paths within the same workflow by path ID alone.

#### Ref fields

The top-level `refs` list defines inline documents that can be referenced by `ref#<id>`
docspecs anywhere a docspec is accepted (in `docs`, `steps`, or route `at` fields).

| Field | Required | Description |
| --- | --- | --- |
| `id` | yes | Unique identifier. Must match `[a-z][a-z0-9-]*`. Referenced as `ref#<id>` in docspecs. |
| `title` | yes | Display title for the document. |
| `body` | yes | The document content (markdown string). Use YAML literal block scalar (` |

Refs are useful for short content that doesn’t warrant its own file: decision criteria,
quick checklists, brief notes, or transitional guidance specific to the signpost
structure.

#### Constraints

- **All IDs** (workflow and path) must be globally unique and match `[a-z][a-z0-9-]*` (valid
  CLI identifiers).
- **Steps** are docspec strings (with optional locspec).
  Must be unique within their parent path.
- **Route `to` targets** must reference existing paths. Format: `path-id` (within current
  workflow) or `workflow-id/path-id` (cross-workflow). Both workflow and path IDs must
  exist. Validated on load.
- **Route `at` targets** must reference existing steps within the target path.
  Validated on load.
- **Ref IDs** must be unique and match `[a-z][a-z0-9-]*`. Local ref docspecs
  (`ref#<id>`) must reference existing entries in `refs`. Validated on load.
- **Docspec prefixes are strict.** Every docspec must start with a recognized prefix:
  `./` (relative file), `ref#` (local ref), `github:` (remote source), or `https://` /
  `http://` (URL). Strings without a recognized prefix are parse errors.
- **File docspecs** (`./` prefix) are resolved relative to the directory containing
  `signposts.yml`. Warn (but don’t fail) on missing files.
  URL and remote docspecs are not validated at load time.
  Locspecs are not validated at load time.
- **Cycles are allowed.** The graph is not required to be a DAG. Cycles are natural in
  workflows where you may revisit earlier paths.
- **Definition order is display order.** Workflows are listed in the output in YAML order;
  paths within each workflow are listed in YAML order.

### `progress.yml` — Per-Project State

Lives in the consuming project (e.g., at the project root alongside `autorust.yaml`).
Committed to git — progress is part of the project record.

```yaml
# progress.yml — Tracks this project's progress through the signpost flow map
signposts: repos/rust-porting-playbook/signposts.yml
started: 2026-02-17
current_path: research

paths:
  assess:
    status: completed
    started: 2026-02-17
    completed: 2026-02-17
    steps:
      ./reference/python-to-rust-playbook.md#dependency-risk-table: completed
      ./reference/python-to-rust-test-coverage-playbook.md: completed
      ref#go-no-go-criteria: completed
    notes: Coverage at 85% on core modules. Proceeding to research.

  research:
    status: in_progress
    started: 2026-02-17
    steps:
      ./reference/python-to-rust-mapping-reference.md: completed
      ./reference/python-to-rust-playbook.md#library-evaluation: in_progress
      ./reference/python-to-rust-playbook.md#library-decisions: not_started
    extra_steps:
      - ./reference/python-to-rust-playbook.md#comrak-vs-pulldown: in_progress
```

#### Schema Reference

**Top-level fields:**

| Field | Required | Description |
| --- | --- | --- |
| `signposts` | yes | Path to the signposts.yml file this progress tracks. |
| `started` | yes | ISO date when progress tracking began. |
| `current_path` | no | Path ID of the currently active path. |
| `paths` | yes | Map of path ID to progress state. |

**Path progress fields:**

| Field | Required | Description |
| --- | --- | --- |
| `status` | yes | `not_started`, `in_progress`, `completed`, or `skipped`. |
| `started` | no | ISO date when work on this path began. |
| `completed` | no | ISO date when this path was completed. |
| `steps` | no | Map of docspec (step identity) to status (`not_started`, `in_progress`, `completed`, `skipped`). |
| `extra_steps` | no | List of project-specific steps discovered during work. Each is a map of docspec to status. |
| `notes` | no | Free-text notes about this path’s progress. |

**Merge semantics:** Steps defined in signposts.yml but not present in progress.yml
default to `not_started`. Extra steps in progress.yml that don’t exist in signposts.yml
are preserved (they are project-specific additions).

### CLI Commands

All commands are under `autorust signposts`. `autorust guide` is a convenience alias
that reads `signposts.yml`.

#### Navigation

**`autorust signposts`** — Overview with workflows, paths, and current position highlighted.

```
$ autorust signposts

Python to Rust Porting Playbook

  Getting Started
  * assess              Understand scope, dependencies, and test coverage    [completed]
    improve-coverage    Grow Python test coverage before porting begins

  Core Workflow
  > research            Evaluate Rust library candidates with real inputs     [in_progress]
    plan                Architecture, module order, feature parity matrix
    setup               Cargo.toml, CI, test fixtures, Python submodule
    port                Port module by module, tests first, leaf to root
    fix                 Cross-validate outputs, categorize diffs, workarounds
    finalize            CLI parity, docs, release configuration
    sync                Track Python upstream updates, manage divergences

  Reference
    mapping-reference   Python-to-Rust type/pattern/construct mapping
    test-coverage       Test coverage strategy and golden testing for porting
    cli-best-practices  Rust CLI best practices and crate recommendations
    code-review         Rust code review checklist

  Guidelines
    porting-rules       Python-to-Rust porting rules and pitfalls
    cli-porting         CLI-specific porting (argparse to clap, SIGPIPE, exit codes)
    rust-general        General Rust coding rules (Edition 2024+)
    rust-cli-patterns   Rust CLI application patterns (clap, error handling, config)
    rust-project-setup  Cargo.toml, CI/CD, lint config, release workflow

  Templates
    observations-template  Template for recording port observations
    triage-template        Template for triaging observations

  Meta
    improving-playbook  How to improve this playbook through your port experience

  * = start    > = current    [status]
```

**`autorust signposts status`** — Progress summary.

```
$ autorust signposts status

Progress: 1/9 workflow paths completed, 1 in progress

  Getting Started
    assess             [completed]    3/3 steps done
    improve-coverage   [not started]

  Core Workflow
    research           [in_progress]  1/3 steps done + 1 extra
    plan               [not started]
    setup              [not started]
    port               [not started]
    fix                [not started]
    finalize           [not started]
    sync               [not started]
```

**`autorust signposts where`** — Current position with context.

```
$ autorust signposts where

Current path: research — Evaluate Rust library candidates with real inputs
  (workflow: Core Workflow)

  Steps:
    [x] ./reference/mapping-reference.md                          Python-to-Rust mapping reference
    [>] ./reference/python-to-rust-playbook.md#library-evaluation  Library evaluation
    [ ] ./reference/python-to-rust-playbook.md#library-decisions   Library decisions
    [>] ./reference/python-to-rust-playbook.md#comrak-vs-pulldown  (extra) Comrak vs pulldown

  Routes:
    -> plan
    -> assess (when: Risk too high, reconsider scope)
```

**`autorust signposts next`** — Suggest what to do next.

```
$ autorust signposts next

Current step: ./reference/python-to-rust-playbook.md#library-evaluation
  (path: research, workflow: Core Workflow)

  When done: autorust signposts done research "./reference/python-to-rust-playbook.md#library-evaluation"
```

#### Content

**`autorust signposts path <id>`** — Show path details with steps, docs, and routes.

**`autorust signposts step <path> <docspec#locspec>`** — Print the doc content for a
specific step. The step’s docspec identifies the file to print.
If the docspec includes a locspec, the relevant section is highlighted.

#### Progress Tracking

**`autorust signposts start <path>`** — Set path status to `in_progress`, set as
`current_path`.

**`autorust signposts done <path> <docspec#locspec>`** — Mark step as `completed`. If
all steps in a path are completed, prompt to mark the path as completed too.

**`autorust signposts skip <path> <docspec#locspec>`** — Mark step as `skipped`.

**`autorust signposts note <path> "<text>"`** — Append a note to a path.

#### Guide Alias

**`autorust guide`** — Lists all paths grouped by workflow with descriptions (flat view, no
progress indicators).

**`autorust guide <path-id>`** — Prints the documentation files for a path.
When a path has multiple docs, each is printed with a header line showing the file path.

### Discovery

`autorust signposts` finds the signposts file by:

1. Look for `signposts` path in `autorust.yaml` config (if present).
2. Walk up from cwd to find the repo root (`.git` directory).
3. Look for `repos/rust-porting-playbook/signposts.yml` relative to that root.
4. If not found, error with: `Signposts file not found.
   Run 'autorust setup' or specify the path in autorust.yaml.`

## Implementation Plan

### Phase 1: Format, Parser, and Navigation (Read-Only)

- [ ] Define `SignpostWorkflow`, `SignpostPath`, `SignpostRoute`, `SignpostsFile`
  dataclasses with validation
- [ ] Implement YAML loader with format version check and constraint validation (unique
  IDs, valid route targets, file existence warnings)
- [ ] Implement `autorust signposts` — workflow/path overview display
- [ ] Implement `autorust signposts path <id>` — path detail display
- [ ] Implement `autorust signposts step <path> <step>` — step doc content display
- [ ] Implement `autorust guide` and `autorust guide <path-id>` — flat listing and doc
  display
- [ ] Write `signposts.yml` for the rust-porting-playbook
- [ ] Write `docs/signposts-format.md` reference documentation
- [ ] Tests: parser validation, display formatting, edge cases (missing files, cycles,
  empty paths)

### Phase 2: Progress Tracking

- [ ] Define `ProgressPath`, `ProgressFile` dataclasses
- [ ] Implement progress.yml read/write with idempotent merge
- [ ] Implement `autorust signposts start <path>`
- [ ] Implement `autorust signposts done <path> <step>`
- [ ] Implement `autorust signposts skip <path> <step>`
- [ ] Implement `autorust signposts note <path> "..."`
- [ ] Implement `autorust signposts status` — progress summary
- [ ] Implement `autorust signposts where` — current position with context
- [ ] Implement `autorust signposts next` — route-based suggestion
- [ ] Tests: state transitions, extra steps, notes, idempotent operations

### Phase 3: Integration and Polish

- [ ] Support `extra_steps` in progress.yml (project-specific steps added during work)
- [ ] Integrate with `autorust setup` — initialize progress.yml when scaffolding a
  project
- [ ] Integrate with `autorust report` — include signpost progress in the summary
  dashboard

## Testing Strategy

- **Unit tests**: Dataclass validation, YAML parsing, constraint checking (unique IDs,
  valid targets), progress merge logic
- **Integration tests**: Load a test signposts.yml, navigate paths, track progress,
  verify state persistence
- **Golden tests**: CLI output for `signposts`, `status`, `where`, `next` commands
  against a known signposts.yml and progress.yml
- **Edge cases**: Cycles in the graph, paths with no steps, paths with no docs, paths
  with no routes, missing files, empty progress.yml

## Decisions Made

1. **Workflows/paths/steps/routes terminology.** The hierarchy is: workflows contain paths,
   paths contain steps and routes.
   This maps naturally to the trail/signpost metaphor and provides the grouping that a
   flat node list lacked.

2. **Explicit IDs on workflows and paths.** IDs are required, globally unique, and stable.
   Titles are display-only and can change freely without breaking route references or
   CLI commands. IDs follow `[a-z][a-z0-9-]*` (valid CLI identifiers).

3. **Optional descriptions on workflows and paths.** Since workflows and paths are structural
   overlay (not docs themselves), they carry their own inline descriptions rather than
   pulling from doc frontmatter.

4. **Steps are docspecs, not IDs.** A step is identified by its `docspec#locspec`
   string. No separate `id` field is needed — the doc reference is the identity.
   Display text comes from the doc’s frontmatter or heading text.

5. **Routes can jump within or across paths.** A route targets a path ID (`to`) and
   optionally a specific step within that path (`at`). This allows intra-path jumps
   (skip ahead, loop back) as well as cross-path transitions.

6. **Routes allow cycles.** The porting playbook naturally has cycles (port -> research
   -> port when you hit a library gap).
   Progress tracking handles this — a path can be revisited.

7. **`when` conditions are descriptive, not enforced.** They are human/agent-readable
   guidance about when to take a route.
   The CLI does not evaluate or gate on them.

8. **progress.yml is committed to git.** Progress is part of the project record, useful
   for handoffs between agents and sessions.

9. **Definition order is display order.** Workflows and paths are displayed in the order
   they appear in the YAML.

10. **Default traversal follows workflow/path order.** Within a workflow, paths are traversed
    in listed order. Routes override this default for conditional jumps.

11. **Local refs for inline content.** Short content that doesn’t warrant its own file
    can be defined in `refs` and referenced via `ref#<id>` docspecs.
    This keeps the signpost structure self-contained for small checklists, decision
    criteria, and transitional notes.

12. **Strict docspec prefixes.** Every docspec must start with a recognized prefix: `./`
    (file), `ref#` (local ref), `github:` (remote), `https://`/`http://` (URL). This
    makes parsing unambiguous and strict — no heuristic guessing.
    The `./` prefix for files avoids YAML quoting issues with `#` and is consistent with
    shell path conventions.

13. **Workflow-level routes for menu navigation.** Workflows can have `docs` and `routes`
    fields, allowing them to serve as decision points or menus. This supports scenarios
    where an agent asks the user questions to determine which workflow to enter (e.g.,
    "Are you starting a new port or syncing an existing one?"). Routes use
    `workflow-id/path-id` format for cross-workflow navigation.

## Open Questions

- **Locspec resolution**: When a docspec includes a locspec (e.g.,
  `file.md#section-name`), should autorust try to extract just that section, or print
  the whole file with a hint?
  Extracting sections from markdown is fragile.
  Initial implementation could print the whole file with a note about which section is
  relevant.

- **Multiple progress files**: Could a project track progress through multiple
  signposts.yml files (e.g., one for porting, one for onboarding)?
  For now, one progress.yml per project.
  Revisit if the need arises.

- **Standalone CLI**: signposts as a concept is general-purpose.
  Should there eventually be a standalone `signposts` CLI separate from autorust?
  For now, implement within autorust.
  Extract later if there’s demand.

## References

- autorust initial version spec:
  `docs/project/specs/active/plan-2026-02-17-autorust-initial-version.md`
- rust-porting-playbook: `repos/rust-porting-playbook/`
- Prior art: XState, Argo Workflows, GitHub Actions, Taskfile.yml, Jupyter Book,
  Runme.dev, Curriculum Network graph.yaml, Ansible playbooks
