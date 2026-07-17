# Contributing to crawl-hunt

Thanks for considering contributing. The single biggest need right now is
**tool coverage** — nobody working on this has deep expertise in every
open-source scanner out there, so pull requests that wire in a new tool as an
additional pipeline phase are the most valuable thing you can contribute.

We are also open to "vibe coders" — contributors who bring creative energy, intuitive ideas, and passion for the project's vision. Your contributions are welcome as long as they align with and advance the core vision of the project (detailed in AGENT.md). Feel free to experiment, propose bold ideas, or iterate playfully, but ensure they stay true to building a comprehensive, CLI-driven, multi-tool security research pipeline.

## Project philosophy (please read before opening a PR)

1. **Rule of enum: never rely on one tool for a given job.** Different
   content-discovery tools, different SAST tools, different DAST tools all
   have blind spots. If you're adding a scanner, it's welcome even if the
   pipeline already has one that does something similar — the goal is
   coverage, not a single "best" tool per category.
2. **Only build on success, not failure.** Source-acquisition phases should
   pull from endpoints that actually returned something (2xx/3xx), not from
   404s or dead paths. If you're adding an acquisition step, filter your
   input accordingly.
3. **Every phase must fail loudly, not silently.** Check for required
   binaries/wordlists/config up front and exit with a clear message before
   the phase starts, rather than letting the tool die mid-run with an
   unhelpful exit code. See `run_phase_timed()` in `site-spider` for the
   existing pattern.
4. **Structured output over raw dumps.** Every phase should ultimately
   produce output in the form `{tool, finding/file, line/endpoint, evidence}`
   wherever the underlying tool supports it (most SAST tools already do
   `file:line`; DAST tools should at minimum report the specific endpoint and
   the specific signature/template that matched). Raw unstructured tool
   output is the exact problem this project exists to get away from — don't
   reintroduce it in a new phase.
5. **Time-bounded by default.** Any phase that could run indefinitely against
   a live target should respect the same time-budget pattern as the existing
   phases (graceful `SIGINT` at the boundary, `SIGKILL` after a short grace
   period) rather than blocking the whole pipeline.
6. **Target-agnostic.** No phase should assume anything about a specific
   target (domain, language, framework). If your addition only makes sense
   conditionally (e.g. a PHP-specific SAST tool), detect the condition first
   (file extensions present in the mirror, etc.) rather than always running
   it.

## Adding a new pipeline phase

Look at the existing phases in `site-spider` as the template. Each phase:

1. Checks its own dependency (binary on `PATH`, required wordlist/config file)
   before the pipeline reaches it — fail with a clear `[!]` message and
   non-zero exit if missing.
2. Runs via the shared `run_phase_timed` pattern: backgrounded, time-budgeted,
   graceful `SIGINT` → `SIGKILL` on timeout, logs to its own file under
   `logs/`.
3. Writes its output under a predictable path inside `OUTPUT_DIR` (e.g.
   `urls/`, `full-mirror/`, or a new subdirectory if the tool's output type
   doesn't fit an existing one) — don't scatter files at the repo root.
4. Documents in the PR description: what the tool adds that existing phases
   don't, and what input it expects (raw target URL? the `urls/all.txt` list?
   the mirrored source tree?).

## Adding correlation logic

If you're working on cross-referencing findings between phases (e.g. matching
a DAST parameter name to a source-code location), keep the matching logic
separate from any individual tool's phase script — it should read already-
produced output files, not be bolted onto a scanner invocation. Include a
short explanation of the matching heuristic used and its known false-
positive/false-negative cases.

## Pull request checklist

- [ ] New phase (or change) documented in `worklog.md`
- [ ] Dependency/config checks fail loudly, not silently
- [ ] Output written to a predictable path, not the repo root
- [ ] Tested against at least two different targets (not just one you had
      open at the time) to confirm it isn't accidentally target-specific
- [ ] No hardcoded scheme/domain/path assumptions

## Questions / proposing a bigger change

If you want to propose something structural (e.g. swapping the correlation
backbone, changing the output format contract) rather than adding a phase,
open an issue first to discuss before writing code — these changes affect
every existing and planned phase, so worth agreeing on the shape before
implementation.