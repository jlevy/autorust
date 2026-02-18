# Feature: autorust Initial Version

**Date:** 2026-02-17 (last updated 2026-02-17)

**Author:** Joshua Levy

**Status:** Draft

## Overview

autorust is a standalone Python CLI tool that helps automate Python-to-Rust porting.
It combines **project scaffolding**, **playbook knowledge injection**, **test
discovery**, **cross-language test mapping**, and **CI-enforceable completeness
checks**.

It generalizes the test mapping system built for the
[flowmark-rs](https://github.com/jlevy/flowmark-rs) port into a reusable tool that works
for any Python-to-Rust port project.

The key insight from the flowmark-rs experience: **tests are the specification for the
port.** Before porting, you need sufficient test coverage.
During porting, you need to track which tests have been ported.
After porting, you need CI to enforce that the mapping stays complete.

**Design philosophy:** autorust does not try to wrap every tool (pytest, cargo, etc.)
itself. Instead, it **orchestrates the workflow and injects the right guidance at the
right time**. The agent (or human) runs the actual tools.
autorust handles the mechanical parts (test discovery, mapping, checking) and surfaces
relevant playbook documentation for the parts that require judgment.

**Installation:** autorust is installed via `uvx autorust` (or `pip install autorust`),
not as a project dependency.
It’s a developer tool, like ruff or pytest.

## Goals

- **Reusable across projects**: No flowmark-specific assumptions.
  All project-specific behavior is driven by configuration.
- **Orchestrator, not wrapper**: Set up project structure, inject relevant playbook
  guidance, and provide mechanical tooling (discovery, mapping, checking).
  Don’t reinvent what pytest, cargo, and agents already do well.
- **Test-readiness workflow**: Surface the right playbook guidance to help projects grow
  test coverage before porting begins.
  For CLI apps, strongly encourage golden testing (tryscript or similar) alongside
  traditional coverage.
- **Cross-language test mapping**: Discover tests on both sides, maintain a mapping, and
  validate completeness — the proven flowmark-rs approach.
- **CI-enforceable**: Every check produces a clear pass/fail exit code suitable for CI
  gating.
- **Agent-friendly**: All artifacts are YAML for readability, diffability, and LLM
  editability. Guidance output is structured for agent consumption.

## Non-Goals

- **Wrapping pytest/cargo**: autorust doesn’t run tests or measure coverage itself.
  It surfaces the right guidance and lets the agent run the appropriate commands.
- **Behavioral equivalence verification**: autorust tracks *structural* test
  correspondence, not whether the Rust test actually tests the same thing.
- **Auto-generating Rust tests from Python**: This is a tracking and assessment tool,
  not a transpiler.
- **Supporting source languages other than Python**: The initial version is
  Python-to-Rust only.
  The architecture should not preclude other source languages but we don’t build for
  them yet.

## Background

### Origin: flowmark-rs Test Mapping System

The flowmark-rs port developed a cross-language test mapping system as the
`flowmark-dev` CLI (in `python/` within that repo).
It proved highly effective:

- Discovered 281 Python tests and 250 Rust tests automatically
- Maintained a hand-curated YAML mapping (202 mapped, 79 excluded, 0 missing)
- Ran as a CI gate — any unmapped test fails the build
- Used idempotent, additive merge so hand-edits are never lost

The system has two main limitations that autorust addresses:

1. **Flowmark-specific**: Test type classification is hard-coded (looks for
   `fill_markdown` calls to classify integration tests).
   The CLI name, default paths, and repo URL are all flowmark-specific.
2. **No setup or guidance**: It assumes the project structure already exists and that
   the Python project already has good test coverage.
   autorust adds project scaffolding and playbook knowledge injection.

### The flowmark-rs Project Layout Pattern

The flowmark-rs project established a pattern that works well:

```
myproject-rs/                    # The Rust port (main repo)
├── repos/
│   ├── myproject/               # Git submodule: original Python source
│   └── rust-porting-playbook/   # Git submodule: playbook with guidance docs
├── port-coverage-mapping/       # autorust artifacts (YAML)
│   ├── python-tests.yaml
│   ├── rust-tests.yaml
│   └── test-mapping.yaml
├── autorust.yaml                # Project configuration
├── src/                         # Rust source
├── tests/                       # Rust tests
└── Cargo.toml
```

This layout gives agents direct access to both the original Python source (for reading
tests and understanding behavior) and the playbook (for guidance at each phase).
`autorust setup` replicates this pattern for any new port.

### The Workflow: Guidance + Tooling

autorust combines two modes:

**Guidance mode** — a topic browser into the playbook:
- `autorust guide` — lists all available guide topics (discovered from the playbook)
- `autorust guide <topic>` — prints the relevant playbook doc(s) for that topic
- `autorust setup` — scaffolds the project, then recommends `autorust guide`

**Tooling mode** — does mechanical work:
- `autorust discover python` — AST-based Python test discovery
- `autorust discover rust` — Cargo-based Rust test discovery
- `autorust mapping init` — creates/updates the mapping skeleton
- `autorust mapping check` — validates mapping completeness (CI gate)
- `autorust report` — summary dashboard

## Design

### Configuration: `autorust.yaml`

All project-specific behavior is driven by a configuration file at the project root:

```yaml
# autorust.yaml — project configuration for autorust

project:
  name: myproject
  # Source Python project (used by `autorust setup` to create submodule)
  python_repo: https://github.com/org/myproject
  python_ref: v1.2.3           # Pinned release tag for test discovery
  # Playbook repo (default: jlevy/rust-porting-playbook; override for forks)
  playbook_repo: https://github.com/jlevy/rust-porting-playbook  # default

# Test type classification heuristics (used by `autorust discover python`)
classification:
  # Function call patterns that indicate integration tests
  integration_indicators:
    - "process_document"
    - "run_pipeline"
  # Files that are infrastructure (not ported)
  infrastructure_files:
    - "test_config.py"
    - "test_cli_file_discovery.py"
  # Files that are golden/fixture tests
  golden_files:
    - "test_ref_docs.py"

# Artifact paths (relative to project root)
paths:
  repos_dir: repos/
  artifacts_dir: port-coverage-mapping/
```

### CLI Commands

```
# Environment validation
autorust validate               # Check that Rust, Cargo, Python, git are
                                # installed and at recent enough versions.
                                # Also run automatically as first step of setup.

# Setup
autorust setup                  # Run validate, then scaffold project: create
                                # autorust.yaml, add submodules (source repo +
                                # playbook), create port-coverage-mapping/ dir.
                                # Recommends `autorust guide` as next step.
                                # --playbook <url> to override the default
                                # playbook repo (jlevy/rust-porting-playbook)

# Guidance (topic browser into the playbook)
autorust guide                  # List all available guide topics
autorust guide <topic>          # Print the playbook doc(s) for a topic

# Test discovery and mapping (mechanical tooling)
autorust discover python        # Discover Python tests → python-tests.yaml
autorust discover rust          # Discover Rust tests → rust-tests.yaml
autorust mapping init           # Create/update test-mapping.yaml skeleton
autorust mapping check          # Validate mapping completeness (CI gate)

# Reporting
autorust report                 # Summary dashboard of porting progress
```

### `autorust validate`

Checks that the environment has the prerequisites for a Python-to-Rust port:

```
$ autorust validate

Checking prerequisites...
  ✓ git 2.44.0 (>= 2.30 required)
  ✓ rustc 1.85.0 (>= 1.80 required)
  ✓ cargo 1.85.0
  ✓ clippy installed
  ✓ rustfmt installed
  ✓ python 3.13.1 (>= 3.11 required)
  ✓ uv 0.10.2

All prerequisites met.
```

If any check fails, autorust prints what’s missing and points to
`autorust guide prerequisites` for installation instructions.
The `prerequisites` guide topic should be added to the playbook as well — it’s useful as
a standalone reference.

Exit code 0 if all checks pass, 1 if any fail.

`autorust setup` runs `validate` automatically as its first step and stops if
prerequisites aren’t met.

### `autorust setup` Flow

1. Prompt for (or read from `autorust.yaml`):
   - Python source repo URL
   - Pinned release tag/ref
   - Playbook repo URL (default: standard playbook)
2. Create `repos/` directory
3. Add Python source as git submodule at `repos/<project-name>/`
4. Add playbook as git submodule at `repos/rust-porting-playbook/`
5. Create `port-coverage-mapping/` directory
6. Write `autorust.yaml`
7. Print: “Setup complete.
   Run `autorust guide` to see available guidance topics.”

### `autorust guide` — Playbook-Driven Topic Browser

**Key design principle:** The playbook owns its own structure via a `docmap.yml` file at
its root. autorust is a thin CLI wrapper that reads this file.
When the playbook adds or reorganizes docs, `docmap.yml` is updated in the playbook repo
and autorust automatically reflects the changes — no autorust code changes needed.

**Playbook discovery:** `autorust guide` finds the playbook by:
1. Walk up from cwd to find the enclosing `.git` directory (repo root).
2. Look for `repos/rust-porting-playbook/` relative to that root.
3. If not found, print an error:
   `Error: Playbook not found at <root>/repos/rust-porting-playbook/`
   `Run 'autorust setup' to add the playbook as a submodule.`
4. Load `repos/rust-porting-playbook/docmap.yml` — this defines all topics.

### `docmap.yml` — Playbook Document Map

Lives at the root of the rust-porting-playbook repo.
Defines the topic hierarchy, descriptions, and file mappings that autorust uses to
generate its `guide` subcommands:

```yaml
# docmap.yml — Document map for the rust-porting-playbook
# This file drives the `autorust guide` CLI.
# Each topic becomes an `autorust guide <topic>` subcommand.

sections:
  - name: Getting Started
    topics:
      - topic: prerequisites
        description: Required tools and installation (Rust, Cargo, Python, uv, git)
        files:
          - reference/prerequisites.md

  - name: Workflow
    topics:
      - topic: playbook
        description: The core 8-phase porting methodology
        files:
          - reference/python-to-rust-playbook.md
      - topic: checklist
        description: Initial port checklist (all phases)
        files:
          - reference/port-checklist-initial-template.md
      - topic: checklist-update
        description: Subsequent update checklist (syncing Python changes)
        files:
          - reference/port-checklist-update-template.md

  - name: Reference
    topics:
      - topic: mapping-reference
        description: Python-to-Rust type/pattern/construct mapping
        files:
          - reference/python-to-rust-mapping-reference.md
      - topic: porting-guide
        description: Detailed porting methodology with code examples
        files:
          - reference/python-to-rust-porting-guide.md
      - topic: test-coverage
        description: Test coverage strategy and golden testing for porting
        files:
          - reference/python-to-rust-test-coverage-playbook.md
          - guidelines/test-coverage-for-porting.md
      - topic: cli-best-practices
        description: Rust CLI best practices and crate recommendations
        files:
          - reference/rust-cli-best-practices.md
      - topic: code-review
        description: Rust code review checklist
        files:
          - reference/rust-code-review-checklist.md

  - name: Guidelines
    description: Compact rules suitable for agent context injection
    topics:
      - topic: porting-rules
        description: Python-to-Rust porting rules and pitfalls
        files:
          - guidelines/python-to-rust-porting-rules.md
      - topic: cli-porting
        description: CLI-specific porting (argparse to clap, SIGPIPE, exit codes)
        files:
          - guidelines/python-to-rust-cli-porting.md
      - topic: rust-general
        description: General Rust coding rules (Edition 2024+)
        files:
          - guidelines/rust-general-rules.md
      - topic: rust-cli-patterns
        description: Rust CLI application patterns (clap, error handling, config)
        files:
          - guidelines/rust-cli-app-patterns.md
      - topic: rust-project-setup
        description: Cargo.toml, CI/CD, lint config, release workflow
        files:
          - guidelines/rust-project-setup.md

  - name: Templates
    topics:
      - topic: observations-template
        description: Template for recording port observations (for playbook feedback)
        files:
          - reference/case-study-observations-template.md
      - topic: triage-template
        description: Template for triaging observations into playbook improvements
        files:
          - reference/case-study-improvement-triage-template.md

  - name: Meta
    topics:
      - topic: improving-playbook
        description: How to improve this playbook through your port
        files:
          - reference/meta-improving-this-playbook.md
```

### How `autorust guide` Uses `docmap.yml`

**`autorust guide` (no topic)** — Loads `docmap.yml`, prints the topic listing:

```
$ autorust guide

Porting playbook guide (from repos/rust-porting-playbook/)

  Getting Started:
    prerequisites         Required tools and installation (Rust, Cargo, Python, uv, git)

  Workflow:
    playbook              The core 8-phase porting methodology
    checklist             Initial port checklist (all phases)
    checklist-update      Subsequent update checklist (syncing Python changes)

  Reference:
    mapping-reference     Python-to-Rust type/pattern/construct mapping
    porting-guide         Detailed porting methodology with code examples
    test-coverage         Test coverage strategy and golden testing for porting
    cli-best-practices    Rust CLI best practices and crate recommendations
    code-review           Rust code review checklist

  Guidelines:
    porting-rules         Python-to-Rust porting rules and pitfalls
    cli-porting           CLI-specific porting (argparse to clap, SIGPIPE, exit codes)
    rust-general          General Rust coding rules (Edition 2024+)
    rust-cli-patterns     Rust CLI application patterns (clap, error handling, config)
    rust-project-setup    Cargo.toml, CI/CD, lint config, release workflow

  Templates:
    observations-template Template for recording port observations
    triage-template       Template for triaging observations into improvements

  Meta:
    improving-playbook    How to improve this playbook through your port

  Run: autorust guide <topic>
```

**`autorust guide <topic>`** — Looks up the topic in the parsed `docmap.yml`, reads the
corresponding file(s) from the playbook submodule, prints their content.
When a topic maps to multiple files (like `test-coverage`), all are printed with clear
separators and file paths.

**Why `docmap.yml` in the playbook, not autorust?** The playbook is the source of truth
for its own documentation structure.
When docs are added, renamed, or reorganized, only `docmap.yml` needs to change —
autorust code stays untouched.
This also makes the playbook self-navigable: anyone browsing the repo can read
`docmap.yml` as a table of contents, independent of autorust.

### Data Model

Identical to the proven flowmark-rs model, with config-driven classification:

**Python test record** (`python-tests.yaml`):
```yaml
- file: tests/test_alerts.py
  function: test_basic_note_alert
  test_type: integration        # Classified by autorust.yaml rules
  line_number: 14
  doc_string: Test basic [!NOTE] alert formatting.
```

**Rust test record** (`rust-tests.yaml`):
```yaml
- file: tests/test_alerts.rs
  function: test_basic_note_alert
  line_number: 8
```

**Mapping record** (`test-mapping.yaml`):
```yaml
- python_file: tests/test_alerts.py
  python_function: test_basic_note_alert
  status: mapped                # mapped | excluded | partial | missing
  rust_file: tests/test_alerts.rs
  rust_function: test_basic_note_alert

- python_file: tests/test_skill.py
  python_function: test_get_skill_content
  python_class: TestGetSkillContent
  status: excluded
  notes: Python skill system infrastructure, not applicable to Rust port.
```

### Identity Keys and Idempotent Merge

This is a core design principle carried over from flowmark-rs.
All discovery and init commands are **idempotent and additive**:

- **Test manifests**: Identity key is `(file, function)` or
  `(file, class_name, function)` for class-based tests.
- **Mapping**: Identity key is `(python_file, python_class, python_function)`.
- **Merge rules**: Auto-discovered records update in place.
  Hand-added entries are never deleted.
  Stale entries are detected by `mapping check`, not by discovery.

### Test Classification

The Python test discovery uses a priority-based classification:

1. If the file matches an `infrastructure_files` pattern → `infrastructure`
2. If the file matches a `golden_files` pattern → `golden`
3. If the function body contains any `integration_indicators` call → `integration`
4. Otherwise → `unit`

This is implemented via AST walking (Python `ast` module), not regex, for reliability.

### Rust Discovery

Two strategies (same as flowmark-rs):

1. **Primary: `cargo test -- --list`** — compiler-authoritative, discovers all `#[test]`
   functions including unit tests in `src/` modules.
2. **Fallback: regex parsing** — for environments without a Rust toolchain.
   Only finds integration tests in `tests/test_*.rs`.

## Implementation Plan

### Phase 1: Core Structure

Set up the package, config, and the `validate` + `setup` + `guide` commands.

- [ ] Define `autorust.yaml` schema as a Python dataclass with validation
- [ ] Implement config loading with sensible defaults
- [ ] Set up CLI with subcommand structure (`validate`, `setup`, `guide`, `discover`,
  `mapping`, `report`)
- [ ] Implement `autorust validate`:
  - Check git, rustc, cargo, clippy, rustfmt, python, uv versions
  - Print clear pass/fail with version numbers
  - On failure, suggest `autorust guide prerequisites`
- [ ] Implement `autorust setup`:
  - Run `validate` as first step; stop if prerequisites not met
  - Interactive prompts for source repo URL, ref, project name
  - Git submodule creation (`repos/<name>/`, `repos/rust-porting-playbook/`)
  - Directory scaffolding (`port-coverage-mapping/`)
  - Write `autorust.yaml`
  - Print: “Run `autorust guide` to see available guidance topics.”
- [ ] Implement `autorust guide`:
  - Playbook discovery: walk up to `.git`, look for `repos/rust-porting-playbook/`
  - Error if not found, suggest `autorust setup`
  - Load `docmap.yml` from playbook root
  - No args: list all topics with descriptions (from docmap)
  - With topic arg: look up files in docmap, print their content
- [ ] Add `docmap.yml` to the playbook repo (draft created at
  `repos/rust-porting-playbook/docmap.yml`; see `docs/docmap-format.md` for format
  specification)
- [ ] Implement docmap loader: parse YAML, validate format version, check topic
  uniqueness, warn on missing files
- [ ] Add `prerequisites` doc to the playbook (`reference/prerequisites.md`)
- [ ] Write tests for validate, setup, and guide commands

### Phase 2: Discovery and Mapping

Port the proven flowmark-rs discovery and mapping engine.

- [ ] Port data models (`PythonTestRecord`, `RustTestRecord`, `MappingRecord`)
- [ ] Port YAML I/O (`yaml_io.py`) — already generic
- [ ] Port Python test discovery — replace hard-coded classification with config-driven
  rules
- [ ] Port Rust test discovery — already generic
- [ ] Port idempotent merge logic — already generic
- [ ] Port mapping checker — already generic
- [ ] Port `mapping init` command — already generic
- [ ] Add `autorust report` command for summary dashboard
- [ ] Write tests using a synthetic fixture project (minimal Python package + minimal
  Rust crate)

### Phase 3: Documentation and Playbook Integration

- [ ] Write README with usage guide and examples
- [ ] Add autorust references to the playbook’s Phase 4 and Phase 5 docs
- [ ] Update playbook test coverage docs to mention autorust
- [ ] Add autorust to the port-checklist templates

## Testing Strategy

- **Unit tests**: Config parsing, classification logic, merge logic, YAML serialization
- **Integration tests**: Using synthetic fixture projects to test the full discover →
  map → check pipeline
- **Golden tests**: For guide command output and report formatting
- **CI**: GitHub Actions running `make lint` and `make test`

## Decisions Made

1. **Standalone repo, installed via `uvx`** (not embedded in the playbook or added as a
   project dependency).
   Published to PyPI. Run with `uvx autorust <command>`.

2. **Python CLI** (not Rust).
   The tool needs to parse Python AST and handle YAML. Python is the natural choice.
   Rust discovery shells out to `cargo test -- --list`.

3. **Orchestrator, not wrapper.** autorust doesn’t try to run pytest or cargo itself for
   coverage/testing. It surfaces the right playbook guidance and lets the agent do the
   work. It only does the mechanical parts: test discovery, mapping, checking.

4. **Submodule-based project layout.** `autorust setup` creates git submodules for the
   source Python repo and the playbook in `repos/`. This gives agents direct access to
   both codebases.

5. **Golden testing emphasis for CLI apps.** For CLI tools, autorust’s guidance strongly
   encourages golden/tryscript testing alongside or even before traditional coverage.
   Golden tests capture end-to-end CLI behavior and serve as the most reliable
   specification for the Rust port.

6. **Configuration-driven classification.** Test type classification rules come from
   `autorust.yaml`, not hard-coded function names.

7. **Same data model as flowmark-rs.** The YAML schema and merge semantics are proven
   and don’t need changes.

## Open Questions

- **Submodule vs clone**: Should `autorust setup` use git submodules (persistent,
  versioned) or shallow clones (simpler, disposable)?
  Submodules match the flowmark-rs pattern and keep the source pinned.

- **Multi-crate Rust projects**: Should Rust discovery support workspaces with multiple
  crates, or just single-crate projects for now?

- **Golden test discovery**: Should `autorust discover python` also discover tryscript
  test files (`.tryscript`, `.sh` fixtures) in addition to pytest tests?
  This would give a more complete picture of test readiness for CLI apps.

## References

- flowmark-rs test mapping system: `python/` directory in
  [flowmark-rs](https://github.com/jlevy/flowmark-rs)
- flowmark-rs test mapping spec:
  `docs/project/specs/active/plan-2026-02-17-test-mapping-meta-test.md`
- rust-porting-playbook: https://github.com/jlevy/rust-porting-playbook
- rust-porting-playbook test coverage docs:
  `reference/python-to-rust-test-coverage-playbook.md`,
  `guidelines/test-coverage-for-porting.md`
- simple-modern-uv template: https://github.com/jlevy/simple-modern-uv
