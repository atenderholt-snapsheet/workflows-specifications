# OpenSpec

This repo stores **OpenSpec** specifications to manage architectural changes and features. Currently, changes are mostly Workflows-centered.

## What is OpenSpec?

[OpenSpec](https://openspec.dev) is a specification-first workflow framework that brings discipline to feature planning and architectural decisions. It creates a clear division between **planning** (what, why, how) and **implementation** (doing the work), enabling teams to align on requirements before writing code.

## Why Specifications as Repository Artifacts?

Specifications checked into your repository have several advantages:

- **Versioned with code**: Specs evolve alongside implementation; you can see why a decision was made by reading git history
- **Team alignment before coding**: Design reviews happen on paper before implementation costs accumulate
- **Context preservation**: Future maintainers understand the original intent and constraints, not just the final code
- **Reviewable artifacts**: Proposals, designs, and tasks become pull request artifacts that the team can discuss and approve
- **Reduced scope creep**: Written acceptance criteria make it clear when a feature is done

## Why OpenSpec?

Reasons to choose OpenSpec:

- **Spec-driven workflow**: Enforces a disciplined flow (propose → review → design → implement → ship) that prevents ad-hoc planning
- **Tooling integration**: Built-in CLI commands make it lightweight to create, validate, and manage changes — no external SaaS
- **Flexible schema**: Extensible to Vice Backend's specific needs (supporting both features and platform changes)
- **Repository-native**: No external system; specifications are durable artifacts that stay with the code

**Comparison to alternatives:**
- **BMAD** (Big Model Architecture Document): Heavier, top-down document with broader scope; OpenSpec is change-focused and incremental
- **SpecKit**: Template-based; OpenSpec adds workflow enforcement and multi-artifact coordination

## Quickstart

### Install the OpenSpec CLI

```bash
npm install -g @openspec/cli
# or, if using Homebrew:
brew install openspec
```

Verify installation:
```bash
openspec --version
```

Get a copy of this store:
```bash
git clone git@github.com:atenderholt-snapsheet/workflows-specifications.git ~/openspec/workflows-specifications
openspec store register ~/openspec/workflows-specifications
```

Install AI skills:
```bash
openspec init
```

### Create Your First Change

1. **Propose a change** — describe what you want to build:
   ```bash
   openspec propose add-new-feature
   ```
   This opens a guided workflow that generates:
   - `proposal.md` — the what and why
   - `specs/<feature>/spec.md` — acceptance criteria and behavior
   - `design.md` — architecture and implementation approach
   - `tasks.md` — actionable steps for the team

2. **Review artifacts** — read the generated files, edit as needed for accuracy

3. **Validate your change**:
   ```bash
   openspec validate --change add-new-feature
   ```

4. **When ready, apply the change** — start implementation:
   ```bash
   openspec apply --change add-new-feature
   ```

5. **Archive when done**:
   ```bash
   openspec archive --change add-new-feature
   ```

## General Workflow

The typical Vice Backend OpenSpec workflow with commands and skills:

| Step                     | What Happens                                                                                 | Command / Skill                                                                                                                            |
|--------------------------|----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| **1. Propose**           | Describe your feature/fix; OpenSpec scaffolds artifacts                                      | `/opsx:propose add-feature-name` or `openspec propose add-feature-name`                                                                    |
| **2. Review**            | Team reads proposal, specs, design in a PR; iterates on artifacts                            | Recommended: create PR with `openspec/changes/<name>/` — post for code review                                                              |
| **3. Design**            | Update design.md, specs, and proposal based on feedback                                      | `/opsx:update <change-name>` or `openspec update --change <name>` to refine artifacts                                                      |
| **4. Apply & implement** | Spec is finalized and approved; marks change ready for development and starts implementation | `/opsx:apply <change-name>` or `openspec apply --change <name>` to start implementation. Recommended: reviewing AI output after each task. |
| **5. Archive**           | Development complete; specs stay in git history for future reference                         | `/opsx:archive <change-name>` or `openspec archive --change <name>` when work ships                                                        |

**Quick reference — Common commands:**

```bash
# Create or explore
/opsx:propose my-feature             # Propose a new change (Cursor skill)
/opsx:explore                         # Browse existing specs (Cursor skill)
openspec list                         # List all changes in flight
openspec show --change <name>         # Show a change's proposal
openspec context                      # Verify OpenSpec root and config

# Review and iterate
/opsx:update my-feature               # Update a change's artifacts (Cursor skill)
openspec status --change <name>       # Check artifact completion status
openspec validate --change <name>     # Validate change structure

# Finalize and implement
/opsx:apply my-feature                # Mark spec as finalized; ready for implementation (Cursor skill)
/opsx:archive my-feature              # Mark as complete after code ships (Cursor skill)
openspec sync-specs                   # Sync specs with code (if configured)
```

Changes are stored in `openspec/changes/` and versioned with the codebase. See [`openspec/`](../openspec) for examples.
