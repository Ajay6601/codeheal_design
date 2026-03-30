# CodeHeal

Autonomous bug detection, diagnosis, and repair agent for Python codebases.

CodeHeal ingests a Python repository, detects bugs across 12 pattern classes using AST analysis and LLM reasoning, traces root causes through cross-file dependency graphs, generates verified fixes with explainable repair reports, and applies them directly creating a git branch with a structured commit.

Tested on production codebases: [httpx](https://github.com/encode/httpx) (16 fixes, $0.003), [click](https://github.com/pallets/click) (15 fixes, $0.001).

---

## Results

| Repository | Files | Bugs Found | Auto-Fixed | Escalated | Cost | Time |
|---|---|---|---|---|---|---|
| httpx (15k stars) | 23 | 23 | 16 | 5 | $0.003 | 157s |
| click (15k stars) | 17 | 26 | 15 | 11 | $0.001 | 114s |
| click (full + LLM) | 62 | 81 | 39 | 21 | $0.103 | 212s |

### What it catches

```
Per-class breakdown (httpx)
┌────────────────────┬──────────┬───────┬──────┐
│ Bug class          │ Detected │ Fixed │ Rate │
├────────────────────┼──────────┼───────┼──────┤
│ missing_return     │       20 │    20 │ 100% │
│ unreachable_code   │        3 │     3 │ 100% │
│ unused_variable    │        5 │     5 │ 100% │
│ missing_null_check │        1 │     1 │ 100% │
│ deprecated_api     │        3 │     2 │  67% │
│ type_mismatch      │        4 │     3 │  75% │
│ circular_import    │        4 │     0 │   0% │
│ resource_leak      │        1 │     0 │   0% │
└────────────────────┴──────────┴───────┴──────┘
```

---

## How It Works

```
Codebase → Ingest → Detect → Route → Diagnose → Fix → Guardrails → Decide → Apply
              │         │        │         │        │        │          │        │
          AST parse  12 bug   3-tier    Root     Patch   Syntax     Confidence  Git
          + import   classes  cost     cause    gen     + scope     gate:       branch
          + call     (AST +   routing  tracing  (AST    + diff     auto-fix    + commit
          graph      LLM)              + LLM    or LLM) + tests    or escalate
```

### Three-Tier Cost Routing

Not every bug needs an LLM. CodeHeal routes each bug to the cheapest fix path that works.

**Tier 1 : AST rules (zero cost):** Unused imports, unused variables, bare except clauses, mutable default arguments, unreachable code, missing return statements, resource leaks. Detection is pattern matching on the AST. Fixes are deterministic transforms. Handles 50-70% of all bugs.

**Tier 2 : Cached patterns (near-zero cost):** When the pattern memory has a high-confidence match from a previous run, skip the LLM and apply the stored fix template. Gets faster over time.

**Tier 3 : Full LLM (pay-per-bug):** Type mismatches, deprecated APIs, missing null checks, complex resource leaks. LLM diagnoses root cause with causal chains and generates a fix with rationale. ~$0.003 per bug with gpt-4o-mini.

### Guardrails

Every fix passes through four checks before it can be applied:

1. **Syntax validation** `ast.parse()` on the patched file
2. **Scope check** fix must only touch code near the bug location
3. **Diff size limit** no fix changes more than 20 lines
4. **Test runner** runs pytest on affected test files; rejects fixes that cause regressions

On Click, 9 fixes were correctly rejected because they broke tests. Zero bad fixes shipped.

### Pattern Memory

After every successful fix, CodeHeal extracts a structured pattern and stores it. On subsequent runs even on different repos matching patterns boost detection confidence and skip expensive LLM calls.

```
120 patterns stored after scanning httpx + click
```

### Explainable Repair Reports

Every fix comes with a structured explanation:

```
[HIGH] missing_null_check at _utils.py:47

  Root cause: proxy_info.get('no', '') may return None.
              Calling .split() on None raises AttributeError.

  Causal chain:
    1. get_environment_proxies() calls getproxies()
    2. proxy_info.get('no', '') may return None
    3. .split() is called without a None check

  Fix: Added null check defaulting to empty string
  Confidence: 0.81 [syntax=1.0 semantic=0.9 tests=1.0 memory=0.0]
  Cost: 1,431 tokens, $0.0005
```

---

## Quick Start

```bash
# Install
pip install -r requirements.txt

# Scan any Python repo (Tier 1 only, zero cost)
python -m codeheal.main --repo /path/to/python/repo --no-llm

# Scan with LLM analysis
export OPENAI_API_KEY=sk-...
python -m codeheal.main --repo /path/to/python/repo

# Apply fixes to files and create a git branch
python -m codeheal.main --repo /path/to/python/repo --apply

# With budget limits
python -m codeheal.main --repo /path/to/repo --max-cost 1.00 --max-llm-calls 30

# Export repair reports
python -m codeheal.main --repo /path/to/repo --output reports.json
```

### Dashboard

```bash
# Terminal 1: API server
python -m codeheal.server

# Terminal 2: Frontend
cd dashboard && npm install && npm run dev

# Open http://localhost:5173
```

The dashboard shows live scan progress, issue list with diff viewer, confidence breakdowns, guardrail results, pattern memory, and full scan logs. Supports both Tier 1 and LLM modes from the UI.


## Detection Classes

| Class | Detection | Fix | Tier |
|---|---|---|---|
| `unused_import` | AST name reference scan | Remove import or specific name | 1 |
| `unused_variable` | AST assign/load analysis per scope | Prefix with underscore | 1 |
| `bare_except` | AST ExceptHandler with no type | Replace with `except Exception:` | 1 |
| `mutable_default_arg` | AST default value type check | `None` default + guard clause | 1 |
| `unreachable_code` | AST statements after return/raise | Remove unreachable lines | 1 |
| `missing_return` | AST path analysis on annotated functions | Insert `return None` | 1 |
| `resource_leak` | AST `open()`/`connect()` without `with` | Wrap in context manager | 1 |
| `type_mismatch` | LLM analysis of function signatures | LLM-generated fix | 3 |
| `missing_null_check` | Heuristic `.get()` → attribute access | LLM-generated null guard | 3 |
| `deprecated_api` | LLM detection of deprecation warnings | LLM-generated migration | 3 |
| `circular_import` | Import graph cycle detection | Escalate (complex refactor) | 3 |
| `broken_call_signature` | Cross-file arg count validation | LLM-generated fix | 3 |

---

## Benchmark on Real Repos

```bash
# Clone and scan httpx
git clone --depth 1 https://github.com/encode/httpx.git tests/httpx_test
python -m codeheal.main --repo tests/httpx_test/httpx --apply

# Clone and scan click
git clone --depth 1 https://github.com/pallets/click.git tests/click_test
python -m codeheal.main --repo tests/click_test/src/click --apply

# See the fixes
cd tests/httpx_test && git diff original codeheal/fixes --stat
```


---

## Tech Stack

- **LangGraph** Agent orchestration with conditional routing
- **Python AST** Deterministic code parsing and analysis
- **OpenAI API** LLM diagnosis and fix generation (gpt-4o-mini)
- **FastAPI** Dashboard backend with live progress streaming
- **React + Vite** Frontend dashboard (Inter + JetBrains Mono)
- **Rich** Colored CLI output

---

## What This Demonstrates

This project mirrors the core architecture of autonomous software repair systems:

**Detect → Diagnose → Fix → Decide → Apply**

The detect-diagnose-fix loop runs as a LangGraph agent pipeline with conditional routing. The three-tier cost system ensures most bugs are fixed at zero LLM cost. The guardrail gate catches bad fixes before they ship. The pattern memory improves over time. The `--apply` flag closes the loop by writing fixes and committing them.

Built by [Ajay Sai Reddy Desireddy](https://github.com/ajaysai) as a proof-of-work project for autonomous code repair engineerin
