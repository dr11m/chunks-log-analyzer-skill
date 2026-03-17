# chunks-log-analyzer-skill

Repository for iterative development of the `chunks-log-analyzer` skill — an orchestrator for parallel log file auditing using AI agents.

This repository contains the complete development lifecycle of the skill: from idea and prototype through iterative testing to a stable version. Each skill version is stored in a separate branch with full history of runs and discovered problems. This approach allows tracking the skill's evolution, understanding which changes led to improvements, and safely experimenting with new approaches.

> The idea evolved from [llm-log-analyzer](https://github.com/dr11m/llm-log-analyzer) — a Python script that used LLM for log analysis. The skill brings this idea to Claude Code/Copilot, adding parallelization via sub-agents and cross-chunk trend analysis.

## What the skill does

The skill analyzes log files of any size by splitting them into chunks and distributing the analysis across multiple parallel agents. Each agent audits its own chunk, then the orchestrator collects results, detects cross-chunk trends and anomalies, and generates a final report.

When you run the skill on a large log file (gigabytes), a problem arises: the model cannot process such volume in a single request. The skill solves this by splitting the log into parts and delegating analysis of each part to a separate agent. The main value is the ability to see the whole picture: how errors are distributed over time, whether there's a trend towards degradation or improvement, and which problems appeared in one chunk and repeated in others.

**How it works:**

1. **Log splitting** — the skill automatically divides the file into chunks of specified size (default ~100KB)
2. **Parallel audit** — each chunk is analyzed by a separate agent that searches for errors, warnings, anomalies
3. **Metrics collection** — the orchestrator collects quantitative data from all agents: how many times each error occurred, when it first and last appeared
4. **Trend detection** — comparing metrics between chunks allows detecting degradation (more errors) or improvement (fewer errors)
5. **Final report** — aggregated result with category details and conclusions about system state

**Key features:**
- Splitting large logs into chunks of specified size
- Parallel auditing of each chunk by a separate agent
- Mandatory output format with exact counts and timestamps
- Cross-chunk metric collection for trend detection (degradation/improvement)
- Output completeness check before submitting results

## Skill Usage

Invoke the skill via `/chunks-log-analyzer` with the path to a log file:

```
/chunks-log-analyzer /path/to/app.log
/chunks-log-analyzer ./logs/production.log
```

The skill automatically:
1. Splits the log into chunks (by size)
2. Runs parallel audit on each chunk
3. Collects metrics and detects cross-chunk trends
4. Generates the final report with conclusions

### Testing

The skill has been tested in two IDEs:

| IDE | Result | Notes |
|-----|--------|-------|
| **Claude Code** | ✅ Most stable | Follows instructions precisely, exact output format |
| **Copilot** | ✅ Reliable | Works consistently, minor formatting issues possible |

Both provide quality auditing. Claude Code is recommended for production tasks.

### References

- [Anthropic: Build Claude Skills](https://docs.anthropic.com/en/docs/claude-code/custom-skills) — official guide for creating skills
- [Copilot: Extend Copilot with skills](https://docs.github.com/en/copilot/extend-copilot/creating-custom-skills-for-copilot) — official Copilot guide

---

## Structure

```
chunks-log-analyzer.md               # current skill (main = stable version)
CHANGELOG.md                    # version history: branch tree, runs, problems
.claude/commands/
  update-skill-changelog.md     # command: audit and update CHANGELOG.md
  record-run.md                 # command: record run + problems
```

## Workflow

1. `main` contains the latest stable version of the skill
2. Each iteration — separate branch: `git checkout -b v2-feature-name`
3. On the branch, edit `chunks-log-analyzer.md`, test, record in `CHANGELOG.md`
4. When stable — merge to `main`

### Commands

**`/update-skill-changelog`** — audit and organize CHANGELOG.md:
- Checks structure of all branches (statuses, parents, links)
- Rebuilds Mermaid diagram and table of contents
- Anonymizes all data (projects → `Project A`, `Project B`, ...)
- Creates new versions, records merges

```
/update-skill-changelog                              # full audit
/update-skill-changelog new-version v2-fix-read      # new branch
/update-skill-changelog merge stable                 # merge to main
```

**`/record-run`** — record skill run:
- Adds row to runs table (model, project, result)
- Adds problems linked to run and model
- Automatically anonymizes projects

```
/record-run sonnet, project X, 5 chunks, SUCCESS, 45 commands
/record-run opus, project Y, FAILED. Problems: agent read log directly
```

## CHANGELOG

Version history and branch tree — in [CHANGELOG.md](CHANGELOG.md).
