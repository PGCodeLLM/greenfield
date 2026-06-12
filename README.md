# Greenfield

**Reverse engineer clean behavioral specs from any codebase.**

Greenfield reads source code, documentation, SDKs, runtime behavior, and binaries, then produces behavioral specifications, test vectors, acceptance criteria, and a full provenance trail. The output describes *what* the software does — not *how* any particular codebase does it — so a fresh implementation team can build against the specs without inheriting the original's internal structure.

Greenfield is a [Claude Code](https://claude.com/claude-code) plugin. It runs inside the `claude` CLI and uses Claude agents as the workers that read code, write specs, and audit output.

## Installation

This repository is both a Claude Code **plugin** and a single-plugin **marketplace** (see `.claude-plugin/marketplace.json`), so you can install it straight from this GitHub fork:

```bash
/plugin marketplace add PGCodeLLM/greenfield
/plugin install greenfield@greenfield-fork
```

Restart Claude Code after installing.

If you previously installed greenfield from the upstream marketplace, remove that copy first — otherwise two plugins named `greenfield` register the same skills:

```bash
/plugin uninstall greenfield@prime-radiant-marketplace
/plugin marketplace remove prime-radiant-marketplace
```

### Pin to a branch, or install from a local clone

```bash
# Pin the marketplace to a specific branch (defaults to the repo's default branch)
/plugin marketplace add PGCodeLLM/greenfield@main

# Local development: add a clone on disk as the marketplace
/plugin marketplace add /path/to/greenfield
/plugin install greenfield@greenfield-fork
```

> **Updating:** `plugin.json` pins `version`, which does not change per commit, so `/plugin update` may not pick up new commits on the fork. After pushing changes, bump `version` in `.claude-plugin/plugin.json`, or simply `/plugin uninstall greenfield@greenfield-fork` and reinstall.

### Upstream

This is a fork of [`prime-radiant-inc/greenfield`](https://github.com/prime-radiant-inc/greenfield), installable from its marketplace:

```bash
/plugin marketplace add prime-radiant-inc/prime-radiant-marketplace
/plugin install greenfield@prime-radiant-marketplace
```

This fork's `/analyze` runs fully autonomously (no confirmation prompts) and stops after Layers 1–4 by default; see Usage below.

## Usage

### Extract specs

```bash
claude
> /analyze /path/to/target   # or just /analyze to analyze the current directory
```

The target path is optional — when omitted, `/analyze` analyzes the current working directory.

`/analyze` runs a seven-layer pipeline, fully autonomously — it never stops to ask which sources, layers, or scope to run. It discovers the available intelligence sources (source, docs, SDK, community, runtime, binary, git history, tests, UI, contracts), gathers evidence from each, synthesizes behavioral specs with provenance citations, then generates test vectors and acceptance criteria. **By default it stops after Layers 1–4**, leaving the raw specs (with full provenance) in `workspace/raw/specs/` as the deliverable. Layers 5–7 (sanitization → second-pass review → fidelity check) are not run automatically; produce source-free specs by running `/sanitize` on the workspace when you need them.

Target shape doesn't matter. Greenfield has been used on single-file minified JavaScript bundles and on source-tree projects; the pipeline also has a path for decompiled native binaries. The methodology adapts to what's there.

Workspace output:

```
workspace/
├── raw/         # Analysis artifacts with source references
├── output/      # Sanitized specs for the implementation team
│   ├── specs/
│   ├── test-vectors/
│   └── validation/
└── provenance/  # Citation audit trail
```

### Run headless (non-interactive, from the shell)

Because `/analyze` runs fully autonomously, you can launch it straight from the shell with Claude Code's print mode (`-p`) instead of opening the TUI:

```bash
IS_SANDBOX=1 claude -p "/greenfield:analyze /path/to/target" \
  --dangerously-skip-permissions \
  --max-turns 1000 \
  --verbose \
  2>&1 | tee ~/greenfield-analyze-$(date +%s).log
```

Omit the path to analyze the current directory:

```bash
IS_SANDBOX=1 claude -p "/greenfield:analyze" --dangerously-skip-permissions --max-turns 1000 --verbose
```

What the flags do, and why they're needed:

| Flag | Why |
|------|-----|
| `-p "…"` | Print mode: run the command once, non-interactively, and exit. |
| `IS_SANDBOX=1` | Marks the environment as a sandbox so `--dangerously-skip-permissions` is permitted (e.g. when running as root / in a container). |
| `--dangerously-skip-permissions` | The pipeline runs lots of varied Bash (git, `find`/`grep`, `npx js-beautify`, optionally `docker`/`podman`, and the target's own test suite) and spawns dozens of subagents. In print mode any un-approved action is denied silently, so a hand-written allowlist is impractical — this skips the prompts entirely. |
| `--max-turns 1000` | The pipeline is many turns (dispatch → commit → gate → remediate → …). A low cap would cut a real run off mid-pipeline; 1000 leaves ample headroom (it still self-terminates at its gates). |
| `--verbose` | Print mode buffers by default; this streams turn-by-turn progress. Add `--output-format stream-json` if you want structured events. |

> ⚠️ **Safety:** `--dangerously-skip-permissions` lets the run execute arbitrary commands — greenfield is "run code to understand code". Only use it on a target you trust, ideally in a throwaway directory, VM, or container. The workspace and its git repo are written into the **current directory**; if the target lives elsewhere, `cd` into a clean working directory first (and add `--add-dir /path/to/target` if the target is outside it).

### Re-sanitize an existing workspace

```bash
claude
> /sanitize /path/to/workspace
```

Re-runs the sanitization pass on an existing workspace. Useful when the initial pass left contamination that the audit caught.

### Implement from the output specs

Greenfield stops at the specs. The implementation team reads `workspace/output/` and builds against the behavioral specs, test vectors, and acceptance criteria there.

## How it works

The pipeline dispatches agents with role-specific prompts across the seven layers. Each dispatch runs one of two generic agent types loaded with a specific skill:

- **analyzer** — the workhorse. Reads evidence and writes specs. Dispatched under many roles across layers: `discovery-agent`, `bundle-splitter`, `chunk-analyzer`, `function-analyzer`, `doc-researcher`, `community-analyst`, `sdk-analyzer`, `cli-explorer`, `behavior-observer`, `feature-discoverer`, `architecture-analyst`, `api-extractor`, `synthesizer`, `module-mapper`, `deep-dive-analyzer`, `behavior-documenter`, `user-journey-analyzer`, `contract-extractor`, `spec-verifier`, `source-completeness-checker`, `test-vector-generator`, `test-generator`, `acceptance-criteria-writer`, `spec-reviewer`, `structural-leakage-reviewer`, `content-contamination-reviewer`, `behavioral-completeness-reviewer`, `deep-read-auditor`, `fidelity-validator`.
- **sanitizer** — rewrites raw specs into output specs free of implementation details. Dispatched during Layer 5 and for remediation during Layers 6 and 7.

### Pipeline

| Layer | Roles | Output |
|-------|-------|--------|
| L1: Intelligence | bundle-splitter, chunk-analyzer, function-analyzer, doc-researcher, community-analyst, sdk-analyzer, cli-explorer, behavior-observer | Raw evidence from source, docs, SDK, community, runtime, binary |
| L2: Synthesis | feature-discoverer, architecture-analyst, api-extractor, synthesizer, module-mapper | Feature inventory, architecture model, module map |
| L3: Deep Docs | deep-dive-analyzer, behavior-documenter, user-journey-analyzer, contract-extractor | Behavioral specs, journeys, contracts |
| Gate 1 | spec-verifier | Correctness, contradictions, gaps |
| Gate 1b | source-completeness-checker | Every user-facing surface captured |
| L4: Validation | test-vector-generator, test-generator, acceptance-criteria-writer | Test vectors, test specs, acceptance criteria |
| Gate 2 | spec-reviewer | Implementation leakage, completeness, quality |
| L5: Sanitization | sanitizer | Output specs rewritten from understanding, not copied |
| L6: Review | structural, content, completeness reviewers + deep-read auditors | Second-pass contamination review |
| L7: Fidelity | fidelity-validators | Flags behavioral detail lost or weakened during sanitization |

## License

Apache 2.0. Copyright 2026 Prime Radiant, Inc.
