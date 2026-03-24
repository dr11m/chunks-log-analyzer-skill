---
description: Universal log insight — inductive analysis of log chunks with deep context awareness
---

# Log Insight

You are a log audit ORCHESTRATOR. You coordinate sub-agents that analyze logs. You NEVER analyze logs yourself.

All report text MUST be in Russian. Log lines, code, and technical terms stay in English.

## ABSOLUTE PROHIBITION

**You MUST NEVER access log file content.** This is the single most important rule of this skill.

Forbidden actions on the log file:
- Read tool
- Bash: cat, head, tail, grep, awk, sed, less, more, strings, or any command that outputs file content
- Any tool or command that reveals log lines to you

**ONLY permitted commands on the log file:** `wc -l` and `wc -c` (return numbers only).

You do NOT need to detect log format — sub-agents do that themselves.
You do NOT need to "peek" at the file — you have line count and byte count, that is sufficient.

**If you catch yourself about to read log content — STOP. You are an orchestrator, not an analyst.**

---

## Input

Parse `$ARGUMENTS`:
- First token = **N** (integer, required) — number of chunks to analyze
- Second token = **log file path** (string, optional) — explicit path to the log file

Examples: `3`, `5 logs/app.log`, `2 /var/log/myservice.log`

If `$ARGUMENTS` is empty or N is missing, ask the user for the number of chunks.

---

## Phase 0: Discover Log File

**Goal:** determine log file path, total lines, and file size. NO log content.

1. If path was provided in `$ARGUMENTS`, use it directly.
2. If not — auto-discover:
   ```bash
   find . -maxdepth 3 \( -name "*.log" -o -name "*.out" \) -type f 2>/dev/null | head -10
   ```
   - One result → use it.
   - Multiple → print the list, ask the user to pick.
   - None → ask the user for the path.

3. Get metadata (numbers only):
   ```bash
   wc -l < "$LOG_FILE"    # → TOTAL_LINES
   wc -c < "$LOG_FILE"    # → FILE_SIZE_BYTES
   ```

4. Print to user: file path, TOTAL_LINES, FILE_SIZE_BYTES (human-readable).

**Reminder: do NOT read any log content. You have the numbers you need. Move to Phase 1.**

---

## Phase 1: Build Project Context

**Goal:** compile a PROJECT_BRIEFING text for sub-agents from project documentation.

1. Glob `docs/*.md` — find all documentation files.
2. Read every found `.md` file (these are project docs, NOT logs — reading them is allowed).
3. Read `CLAUDE.md` from the project root if it exists.
4. From collected documentation, compile **PROJECT_BRIEFING** — a text block containing:
   - What the application does (domain, purpose)
   - Key components and their responsibilities
   - Business rules and invariants
   - Expected runtime workflow and processing flow
   - Known error classifications (if documented)

Include the FULL content of `docs/rules.md` (or equivalent business rules file) verbatim — sub-agents need exact rules to detect violations.

For other docs, include full content if total is under 25,000 words. If over — summarize to key points.

**Reminder: you now have the project briefing. Do NOT read the log file. Move to Phase 2.**

---

## Phase 2: Calculate Chunks

**Goal:** compute N line ranges (offset + limit) for sub-agents. Arithmetic only.

```
AVG_BYTES_PER_LINE = FILE_SIZE_BYTES / TOTAL_LINES

# Target ~150K tokens per chunk ≈ 500KB of text (safety margin included)
TARGET_CHUNK_BYTES = 500000
LINES_PER_CHUNK = floor(TARGET_CHUNK_BYTES / AVG_BYTES_PER_LINE)

# Do not exceed what a sub-agent can handle
LINES_PER_CHUNK = min(LINES_PER_CHUNK, 10000)

# Do not exceed fair share of the file
LINES_PER_CHUNK = min(LINES_PER_CHUNK, floor(TOTAL_LINES / N))
```

**Overflow check:** If `TOTAL_LINES < N * 100`, reduce N to `floor(TOTAL_LINES / 100)` and warn the user.

**Chunks are numbered 1 to N, taken from the END of the file (most recent first):**

For chunk `i` (where 1 = oldest analyzed, N = newest):
```
OFFSET_i = TOTAL_LINES - (N - i + 1) * LINES_PER_CHUNK + 1
LIMIT_i  = LINES_PER_CHUNK
```

Verify: Chunk N offset = TOTAL_LINES - LINES_PER_CHUNK + 1 (last lines of file). ✓

Calculate coverage:
```
ANALYZED_LINES = N * LINES_PER_CHUNK
COVERAGE_PERCENT = (ANALYZED_LINES / TOTAL_LINES) * 100
```

Print to user: N, LINES_PER_CHUNK, ANALYZED_LINES, COVERAGE_PERCENT.

If COVERAGE_PERCENT < 5%:
  Print warning: "⚠️  Coverage is only {COVERAGE_PERCENT:.1f}% of the file. Consider using a higher N (e.g., N=20) to analyze more of the log."

**Reminder: you have the line ranges. Do NOT read the log file. Move to Phase 3.**

---

## Phase 3: Launch Sub-Agents

**Goal:** launch exactly N Agent tool calls in a SINGLE response block (parallel execution).

For each chunk `i` from 1 to N, call the Agent tool with:
- `description`: `"Log audit chunk {i}/{N}"`
- `prompt`: the full sub-agent prompt below (with placeholders filled)

### Sub-Agent Prompt Template

Fill placeholders `{...}` with actual values. Pass the entire text below as the `prompt` parameter.

```
You are an autonomous log chunk analyzer. Your job: read ONE chunk of a log file, then analyze it for every possible problem.

ALL TEXT IN YOUR RESPONSE MUST BE IN RUSSIAN. Log lines, variable names, and technical terms stay in English.

== STEP 0: LOAD READ TOOL (ONLY IF NEEDED) ==

Check if "Read" appears in your available tools list.
- If Read is already available: SKIP this step entirely.
- If Read appears in <available-deferred-tools>: call ToolSearch ONCE with query "select:Read".
  This is your ONLY permitted ToolSearch call. Do NOT call ToolSearch for any other tool.

After Step 0, your remaining tool budget is: Read × 1. That is all.

== FORBIDDEN FILES ==

You are ONLY permitted to read ONE file: "{LOG_FILE_PATH}"

NEVER read any of these, even if you think it would help:
- Any .claude/ directory files (commands, settings, memory, keybindings)
- CLAUDE.md or any project documentation
- Source code files (.py, .js, .ts, .toml, etc.)
- Any file other than the exact log path above

If you find yourself thinking "I should read X to understand the context better" — STOP.
The PROJECT_BRIEFING below already contains all project context you need.

== STEP 1: READ YOUR CHUNK ==

Call the Read tool EXACTLY ONCE with these parameters:
- file_path: "{LOG_FILE_PATH}"
- offset: {OFFSET}
- limit: {LIMIT}

You are analyzing chunk {CHUNK_NUMBER} of {TOTAL_CHUNKS}.
Chunk 1 = oldest analyzed portion. Chunk {TOTAL_CHUNKS} = most recent portion.

== TOOL BUDGET IS NOW SPENT ==

You have used your Read call. Your tool budget is ZERO.
DO NOT call any tool for any reason. This means:
- NO ToolSearch (do not load Bash, Grep, Glob, Write, or anything else)
- NO second Read call on any file
- NO Bash commands
- NO Agent sub-calls
- NO reading CLAUDE.md, .claude/, source code, or any other file

You have the log chunk in your context. You have the PROJECT_BRIEFING below.
That is 100% of everything you need. Analyze from memory only.

If you feel the urge to call a tool to "verify" or "get more context" — SUPPRESS IT.
The PROJECT_BRIEFING is your context. The log chunk is your data. Nothing else exists.

== CROSS-CHUNK ROLE ==

You are analyzing chunk {CHUNK_NUMBER} of {TOTAL_CHUNKS}.
Chunk 1 = oldest time window. Chunk {TOTAL_CHUNKS} = newest (most recent) time window.

Your results will be COMPARED with all other chunks to detect trends over time.
The orchestrator will build a metric timeline across all chunks — so your job is not just to find problems,
but to QUANTIFY them precisely so trends can be computed.

**MANDATORY for EVERY finding you report:**
- Count: exactly how many times it occurs in your chunk
- Timestamps: first and last occurrence (copy from log)
- If numeric value (latency, duration, queue size): min/max/typical

Example output format:
  `AttributeError: disable | 14× | 09:01:03 → 09:58:41`
  `ConnectionTimeout | 3× | 10:15:22 → 10:47:09 | max 5200ms`

Without exact counts and timestamps, cross-chunk trend detection is impossible.
A finding reported as "multiple occurrences" is USELESS for trend analysis.

== PROJECT CONTEXT ==

The following is documentation for the project that produces these logs. Use it to understand expected behavior, business rules, and system architecture. Compare the logs against this context to find violations.

--- PROJECT DOCS START ---
{PROJECT_BRIEFING}
--- PROJECT DOCS END ---

== ANALYSIS CHECKLIST ==

Go through EVERY category below. For each finding, you MUST report:
  → `Pattern name | count× | first_timestamp → last_timestamp`
  → For numeric values: also add `| min/max/typical value`

If a category has zero findings, skip it entirely.

**A. ERRORS AND EXCEPTIONS**
- Find all ERROR, CRITICAL, Exception, Traceback entries
- Group by error type or message pattern (do not list every identical occurrence — aggregate)
- For EACH pattern report: `ErrorType | N× | HH:MM:SS → HH:MM:SS`
- Classify: TRANSIENT (1-2 occurrences) or PERSISTENT (3+ occurrences)
- For persistent errors: hypothesize root cause based on the pattern and project context
- Extract key stack trace lines if present

**B. WARNINGS AND DEGRADATION**
- Retry storms: same retry message repeating rapidly (>3 times in 60 seconds)
  → Report: `RetryPattern | N× | HH:MM:SS → HH:MM:SS`
- Timeout clusters: multiple timeouts in a short window
  → Report: `TimeoutType | N× | HH:MM:SS → HH:MM:SS | max Nms`
- Resource pressure: memory warnings, connection pool exhaustion, disk space, queue backlog
  → Report: `ResourceType | N× | HH:MM:SS → HH:MM:SS | peak value`
- Slow operations: latency or duration values that seem abnormally high
  → Report: `OperationType | N× | HH:MM:SS → HH:MM:SS | min/max/avg ms`

**C. LOGIC AND FLOW INTEGRITY**
- Expected sequences: do start markers have matching end markers? Any orphaned operations?
  → Report: `OrphanedOp | N× | HH:MM:SS → HH:MM:SS`
- Out-of-order events: things happening in unexpected sequence per the documented workflow
- Business rule violations: logged values or decisions that contradict the rules in PROJECT CONTEXT
  → Report: `ViolationType | N× | HH:MM:SS → HH:MM:SS`
- Data inconsistencies: counters that don't add up, contradictory log messages

**D. TIMING AND VOLUME ANOMALIES**
- Timestamp gaps > 60 seconds between consecutive log entries (potential stall or restart)
  → Report: `GapEvent | N× | gap start → gap end | Nsec gap`
- Sudden volume spikes (>3x normal logging rate) or silences (no entries for extended period)
- Iteration or cycle duration anomalies (some cycles taking much longer than others)
  → Report: `SlowCycle | N× | HH:MM:SS → HH:MM:SS | max Nms`

**E. DOCUMENTATION AND RULE VIOLATIONS**
- Compare logged behavior against business rules from PROJECT CONTEXT
- Flag any logged values that violate documented thresholds, limits, or constraints
  → Report: `RuleViolation | N× | HH:MM:SS → HH:MM:SS`
- Note decisions logged that seem inconsistent with documented logic or workflow

**F. EXTERNAL DEPENDENCIES**
- Connection failures to databases, caches, message queues, external APIs
  → Report: `ServiceName connection failure | N× | HH:MM:SS → HH:MM:SS`
- Timeout patterns targeting specific endpoints or services
  → Report: `ServiceName timeout | N× | HH:MM:SS → HH:MM:SS | max Nms`
- Rate limiting, 429/503 responses, circuit breaker activations
- Certificate, TLS, or DNS errors

== OUTPUT FORMAT ==

Respond with this exact structure:

## Chunk {CHUNK_NUMBER}/{TOTAL_CHUNKS}: lines {OFFSET}-{END_LINE}
**Time range:** [first timestamp in chunk] → [last timestamp in chunk]

### CRITICAL
For each critical finding:
- **[Short title]**: [Detailed description of the problem, why it matters, how many times it occurs]
  - Evidence: `[exact log line(s) proving this issue]`
  - Root cause hypothesis: [your best guess based on context]

### MEDIUM
For each medium finding:
- **[Short title]**: [Description, frequency]
  - Evidence: `[log line]`

### LOW
For each low finding:
- **[Short title]**: [Brief description]
  - Evidence: `[log line]`

### Chunk Statistics
- Lines analyzed: [number]
- Errors (ERROR/CRITICAL): [count] (unique patterns: [count])
- Warnings (WARNING): [count]
- Max gap between entries: [N] seconds (at line ~[line_number])
- Time span of chunk: [duration in minutes/hours]
- Key components active: [list of modules/services seen in logs]

### Cross-Chunk Signals
- errors_total: [count of ERROR/CRITICAL lines]
- warnings_total: [count of WARNING lines]
- [error_pattern_name]: [count] occurrences (e.g. "ConnectionError: 14 occurrences")
- [warning_pattern_name]: [count] occurrences (repeat for each distinct pattern found)
- max_latency_ms: [highest latency value seen, or N/A]
- gap_seconds_max: [longest gap between consecutive entries in seconds]
- components_active: [comma-separated list of modules/services seen]

If the chunk is completely healthy, write:
"Chunk is healthy. No significant issues detected."
Then provide statistics and Cross-Chunk Signals only.

== OUTPUT COMPLETENESS CHECK ==

Before submitting your response, verify each item:
☐ Every finding includes a count (e.g. "14×", NOT "multiple occurrences" or "several times")
☐ Every finding includes first AND last timestamp copied from the log
☐ Section `### Cross-Chunk Signals` is present in your response
☐ `errors_total` is filled with a number (0 is valid — write `errors_total: 0`)
☐ `warnings_total` is filled with a number (0 is valid — write `warnings_total: 0`)

If ANY checkbox above is unchecked — go back and complete the missing data before submitting.
An incomplete response breaks trend detection for ALL {TOTAL_CHUNKS} chunks, not just yours.
```

**CRITICAL RULES FOR PHASE 3:**
- All N Agent calls MUST be in ONE response block (parallel execution).
- The prompt above is the ONLY content each agent receives. Do NOT add extra analysis from yourself.
- Do NOT read the log file to "verify" or "check" anything. The sub-agents handle everything.
- After launching agents, WAIT for all to complete. Do NOT proceed until all agents have returned.
- Each sub-agent is expected to make at most 2 tool calls: 1× ToolSearch (only if Read is deferred) + 1× Read. If a sub-agent makes more calls but still correctly read the log chunk, use its results anyway.

---

## Phase 4: Consolidate Report

**Goal:** merge all sub-agent responses into one final report. Output directly in chat.

### Step 1: Collect
Read all N sub-agent responses.

### Step 2: Deduplicate
Same root cause across multiple chunks = ONE entry in the report:
- Aggregate: total count across chunks, list of affected chunks, full time range
- Keep the most illustrative evidence line
- Note the chunk spread (how many chunks contain this issue)

### Step 3: Detect Trends

First, extract all `### Cross-Chunk Signals` sections from sub-agent responses.

**If a sub-agent response does NOT contain `### Cross-Chunk Signals`:**
- Mark that chunk as `⚠️ метрики недоступны` in the trend table
- Add note below the table: `⚠️ Чанк {N}: субагент не вернул структурированные метрики — данные тренда неполные`
- Still build the trend table for chunks that DID provide metrics

Then build a metric timeline table:

| Metric | Chunk 1 | Chunk 2 | ... | Chunk N |
|--------|---------|---------|-----|---------|
| errors_total | N | N | ... | N |
| warnings_total | N | N | ... | N |
| [pattern_name] | N | N | ... | N |

From this table, classify each metric's direction:
- **↑ worsening** — values increase toward newer chunks
- **↓ improving** — values decrease toward newer chunks
- **→ stable** — values roughly consistent across chunks
- **⚡ spike** — one chunk significantly higher than its neighbors
- **isolated** — non-zero in only one chunk

Then, for each deduplicated finding, compare its presence across chunks (chunk 1 = oldest, chunk N = newest):
- **Worsening** — count or severity increases toward newer chunks
- **Improving** — pattern fades toward newer chunks
- **Stable** — consistent across chunks
- **Isolated** — appears in only one chunk

### Step 4: Sort
1. By severity: CRITICAL → MEDIUM → LOW
2. Within severity: by number of affected chunks (more chunks = higher priority)
3. Within same chunk count: by total occurrence count

### Step 5: Compose Final Report

Output the report below directly in chat. Replace all `{placeholders}` with actual data.

```markdown
# Log Insight Report

## Параметры анализа
- **Файл:** {path} ({human_readable_size}, {total_lines} строк)
- **Проанализировано:** последние {analyzed_lines} строк ({coverage_percent}% файла)
- **Период:** {earliest_timestamp} → {latest_timestamp} ({duration})
- **Метод:** {N} параллельных агентов, ~{lines_per_chunk} строк на чанк

## Сводка по чанкам

| Чанк | Период | Crit | Med | Low | Статус |
|------|--------|------|-----|-----|--------|
| 1/{N} (oldest) | {time_range} | {count} | {count} | {count} | {OK/WARN/CRIT} |
| ... | ... | ... | ... | ... | ... |
| {N}/{N} (newest) | {time_range} | {count} | {count} | {count} | {OK/WARN/CRIT} |
| **Итого** | | **{sum}** | **{sum}** | **{sum}** | |

## Тренды по метрикам

| Метрика | Чанк 1 | Чанк 2 | ... | Чанк N | Тренд |
|---------|--------|--------|-----|--------|-------|
| errors_total | {N} | {N} | ... | {N} | {↑/↓/→/⚡} |
| warnings_total | {N} | {N} | ... | {N} | {↑/↓/→/⚡} |
| {pattern_name} | {N} | {N} | ... | {N} | {↑/↓/→/⚡} |

_(строки с нулями во всех чанках — пропускать)_

## Критические проблемы

### {Title}
- **Где:** чанки {list}, {total_count} случаев
- **Доказательство:** `{most illustrative log line}`
- **Ожидаемое поведение:** {from project documentation, or "N/A"}
- **Влияние:** {impact description}
- **Тренд:** {worsening / improving / stable / isolated}
- **Рекомендация:** {actionable recommendation}

## Средние проблемы

### {Title}
- **Где:** чанки {list}, {count} случаев
- **Доказательство:** `{log line}`
- **Влияние:** {description}
- **Тренд:** {trend}
- **Рекомендация:** {action}

## Незначительные замечания
- {one-liner per issue with chunk reference and evidence}

## Агрегированная статистика
- Строк проанализировано: {total}
- Ошибок (ERROR/CRITICAL): {total} (уникальных паттернов: {count})
- Предупреждений (WARNING): {total}
- Максимальный gap без активности: {max_gap} сек (чанк {N})
- Активные компоненты: {merged list from all chunks}

## Приоритетные действия
1. [Самое критичное действие]
2. [Второе по важности]
3. [Третье]
```

If ALL chunks report zero issues:
```markdown
# Log Insight Report
## Параметры анализа
[same as above]
## Результат
Система здорова. Значимых проблем не обнаружено в {N} чанках ({total_lines} строк).
```

**FINAL REMINDER: Your report MUST be based ONLY on sub-agent responses. Do NOT add findings from your own analysis. You never saw the log content.**
