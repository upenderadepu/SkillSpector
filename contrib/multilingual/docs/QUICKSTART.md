# Quickstart Guide

## Prerequisites

```bash
# Activate the virtual environment
source .venv/bin/activate

# Verify SkillSpector works
skillspector scan ./tests/fixtures/malicious_skill/ --no-llm
```

Set up API keys for LLM mode (`.env` at repo root):

```bash
# Single key (standard OpenAI-compatible)
OPENAI_API_KEY=sk-or-xxxxxxxxxxxxxxxxxxxxxxxx

# Multi-key pool (recommended for batch)
SKILLSPECTOR_API_KEYS="
sk-or-xxx1|https://api.deepseek.com/v1|deepseek-v4-flash
sk-or-xxx2|https://api.deepseek.com/v1|deepseek-v4-flash
...
"

# Active provider
SKILLSPECTOR_PROVIDER=openai
SKILLSPECTOR_MODEL=deepseek-v4-flash
```

## Basic Usage

### Static-only batch (fastest, no API keys needed)

```bash
python -m contrib.multilingual.batch_scan ./skills/ --no-llm
```

Scans all skills in `./skills/`, terminal output, 4 workers. ~0.1s per skill.

### Full LLM batch

```bash
python -m contrib.multilingual.batch_scan ./skills/ -f terminal --workers 4
```

Same but with LLM semantic analysis. ~5-30s per skill depending on file count.

### Test with the built-in fixtures

```bash
# Static mode (sub-second)
python -m contrib.multilingual.batch_scan ./tests/fixtures/ -f terminal --workers 4 --no-llm

# LLM mode (~3 min with 7 workers)
python -m contrib.multilingual.batch_scan ./tests/fixtures/ -f terminal --workers 7
```

23 skills, designed to test every detection rule.

## Output Formats

```bash
# Terminal (default) — human-readable table with colors
python -m contrib.multilingual.batch_scan ./skills/ -f terminal

# JSON — machine-readable, good for CI pipelines
python -m contrib.multilingual.batch_scan ./skills/ -f json -o report.json

# Markdown — good for PR comments, docs
python -m contrib.multilingual.batch_scan ./skills/ -f markdown -o report.md
```

## Tuning Workers

| Scenario | --workers | Why |
|----------|-----------|-----|
| Free-tier API key | 1 | Avoid 429 rate limits |
| Paid basic tier | 4 (default) | Good balance |
| Enterprise / multi-key | 7-10 | Maximize throughput |
| Debugging | 1 | Sequential output, easier to read |

```bash
# Single worker for debugging
python -m contrib.multilingual.batch_scan ./skills/ --workers 1 -V

# Verbose mode shows debug logs
python -m contrib.multilingual.batch_scan ./skills/ --workers 4 -V
```

## Language Options

```bash
# Auto-detect (default) — uses Unicode script ratio
python -m contrib.multilingual.batch_scan ./skills/ --lang auto

# Force a specific language
python -m contrib.multilingual.batch_scan ./skills/ --lang zh

# Available: auto, en, zh, ja, ko
```

For non-English skills, the scanner automatically applies LLM gap-fill for 8 vulnerability rules that static English-keyword patterns cannot detect.

```bash
# Disable LLM requirement for non-English (results may be incomplete)
python -m contrib.multilingual.batch_scan ./skills/ --no-require-llm --no-llm
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All skills safe (no HIGH/CRITICAL) |
| 1 | At least one skill has HIGH or CRITICAL risk |
| 2 | Scan errors occurred (timeouts, crashes) |

Useful for CI:

```bash
python -m contrib.multilingual.batch_scan ./skills/ -f json -o report.json
if [ $? -eq 0 ]; then
    echo "All clean"
fi
```

## Quick Comparison: Upstream vs Batch

```bash
# Upstream — scan one skill
skillspector scan ./skills/my-skill/ -f json -o upstream.json

# Batch — scan all skills
python -m contrib.multilingual.batch_scan ./skills/ -f json -o batch.json

# Diff the results for any skill
# batch.json.skills[*].scan_mode = "multilingual-enhanced"
# batch.json.skills[*].enhancements = {...}
```

Key differences in batch output:
- `scan_mode: "multilingual-enhanced"` — provenance marker
- `enhancements.gap_fill_applied` — true if LLM gap-fill was used
- `enhancements.english_keyword_rules_skipped` — count of static rules bypassed
- `skill.language` — detected language tag

## Troubleshooting

### "No LLM API key configured"
Either set up `.env` with API keys, or use `--no-llm` for static-only mode.

### Connection errors during LLM scan
The scanner has built-in HTTP timeouts (8s connect, 30s read). Failed skills are marked as errors and other workers continue. Reduce `--workers` if rate limits appear.

### "Event loop is closed" warnings
Harmless. Suppressed by Patch 7. Does not affect results.

### Skills timing out (90s limit)
A skill that takes >90s is marked as timeout and skipped. Increase `--workers` to overlap more skills, or check network connectivity to the LLM provider.

### WARNING: model_info token limit
Harmless. Add your model to `model_registry.yaml` if you want accurate token budgeting. Otherwise a 128K default is used.
