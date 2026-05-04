# SESSION LOG — Stem Agent

## Session 1 — 2026-05-04

### What was built
- Full project architecture designed and initialized on disk
- Git repo created locally and pushed to GitHub
- Virtual environment set up at `.venv/` with all dependencies installed
- All files and folders created (empty, ready to fill)

### Decisions made

**Domain:** Code QA / bug detection. One domain only, go deep.

**Language & stack:** Python, raw OpenAI API calls, no LangChain.
Model: `gpt-4o`, JSON mode for all structured LLM responses.

**API key:** `.env` file with `python-dotenv`. Never committed.
`.env.example` committed as the template.

**Benchmarks:** Both — hardcoded fixtures in `benchmarks/problems/`
committed to repo, plus LLM-generated ones at runtime via `benchmarks/generator.py`.

**Interface:** `main.py` as entrypoint, delegates to `stem_agent/cli.py`
which uses argparse. Flags planned: `--task`, `--compare`, `--verbose`.

**Stage 1 SENSE:** One LLM call, result cached to `cache/domain_knowledge.json`.
Never repeated. Prompt covers seven dimensions:
  1. bug_taxonomy
  2. test_design_rules
  3. static_analysis (mypy, flake8, bandit, pylint — with run_command + output_format)
  4. failure_heuristics (failure signature → bug category + follow-up question)
  5. code_smells (textual patterns that predict bugs before running anything)
  6. severity_levels (critical / high / medium / low + report_action)
  7. workflow (ordered QA steps with feeds_into)

**Stage 2 DIFFERENTIATE:** One LLM call. Consumes all 7 dimensions from Stage 1.
Generates `cache/blueprint.json` containing:
  system_prompt, tool_sequence, workflow_steps, benchmark_criteria, pass_threshold.
The Stage 1 python_examples get embedded as few-shot examples inside the system prompt.

**Stage 3 VALIDATE:** Loop over benchmark problems.
  - Type B failure (execution) → tight inner loop, retry with error context, cheap.
  - Type A failure (structural) → wide outer loop, back to Stage 2, expensive.
  Exit when score >= pass_threshold.

**Stage 4 EXECUTE:** Runs as the specialized agent. No longer a stem agent.
Outputs structured bug report to `outputs/bugs_report.json`.
`--compare` flag runs a bare GPT-4o call on the same code for before/after demo.

### File architecture
See `stem_agent_discussion.md` for the full tree.
Key layout:
  stem_agent/          ← core package
    stage1_sense.py
    stage2_differentiate.py
    stage3_validate.py
    stage4_execute.py
    prompts/           ← all LLM prompt strings, never inline
    benchmarks/        ← fixtures.py (hardcoded), generator.py (LLM), runner.py
    benchmarks/problems/  ← bank_transfer.py, list_processor.py, user_auth.py
    tools/             ← run_pytest.py, run_mypy.py, run_flake8.py, run_bandit.py, file_io.py
  cache/               ← gitignored, runtime LLM outputs
  outputs/             ← gitignored, final bug reports
  tests/               ← test_sense.py, test_differentiate.py, test_benchmarks.py

### Next session — start here
Fill these two files first, in this order:
  1. stem_agent/config.py  — loads .env, exposes constants
  2. stem_agent/client.py  — single OpenAI client instance everything imports

Then move to stem_agent/prompts/sense_prompt.py and stem_agent/stage1_sense.py.