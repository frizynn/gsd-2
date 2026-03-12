You are executing GSD auto-mode.

## UNIT: Research Project (triggered by Milestone {{milestoneId}} "{{milestoneTitle}}")

All relevant context has been preloaded below — start working immediately without re-reading these files.

{{inlinedContext}}

## Task

Conduct comprehensive project-level research by spawning 4 parallel researchers,
then synthesizing their findings into a unified summary.

### Step 1: Read Templates

Read the 5 template files from `~/.gsd/agent/extensions/gsd/templates/research-project/` to understand
the expected output format for each domain:
- `STACK.md`
- `FEATURES.md`
- `ARCHITECTURE.md`
- `PITFALLS.md`
- `SUMMARY.md`

### Step 2: Parallel Research (4 agents)

Use the Agent tool to spawn 4 workers IN PARALLEL (single message, 4 tool calls).
Each writes one file to `{{outputDir}}/`:

**Agent 1 — Stack Research** → `{{outputDir}}/STACK.md`
- Research technologies, versions, libraries for this domain
- Use `resolve_library` / `get_library_docs` for verification
- Follow STACK.md template structure

**Agent 2 — Features Research** → `{{outputDir}}/FEATURES.md`
- Research table stakes, differentiators, anti-features
- Analyze feature dependencies and MVP definition
- Follow FEATURES.md template structure

**Agent 3 — Architecture Research** → `{{outputDir}}/ARCHITECTURE.md`
- Research system patterns, project structure, data flows
- Include code examples for key patterns
- Follow ARCHITECTURE.md template structure

**Agent 4 — Pitfalls Research** → `{{outputDir}}/PITFALLS.md`
- Research common mistakes, tech debt patterns, security issues
- Map pitfalls to likely roadmap phases
- Follow PITFALLS.md template structure

Each agent receives: project description, requirements, milestone context, and its
specific template content. Each uses web search and library docs to verify findings.

### Step 3: Synthesis

After all 4 complete, read the 4 output files. Then write `{{outputDir}}/SUMMARY.md`
following the SUMMARY.md template: synthesize (don't concatenate), derive roadmap
implications, assess confidence, identify research flags.

### Step 4: Commit

Git add and commit all files in `{{outputDir}}/`.

**Skill Discovery ({{skillDiscoveryMode}}):**{{skillDiscoveryInstructions}}

You MUST ensure `{{outputAbsDir}}/SUMMARY.md` exists before finishing.

When done, say: "Project research complete."
