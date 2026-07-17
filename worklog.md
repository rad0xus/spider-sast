# Worklog

A running log of what's been built, what broke, and why — kept so
contributors (and future-us) don't have to reverse-engineer decisions from
git blame alone.

---

## site-spider: v1 → v15 (initial build)

**Goal:** a single command that takes a target and produces (a) a list of
discovered URLs and (b) as much of the site's source code as can be mirrored,
ready to hand to a SAST tool.

### v1–v11 — progress bar iterations
Early versions used a fake percent-complete progress bar driven by fixed
checkpoints (e.g. jump to 10%, run a tool, jump to 40%). Problem: the bar sat
frozen for the entire runtime of whatever tool was actually running, since
there was no way to know real completion percentage for an open-ended crawl —
it wasn't actually tracking anything, just decorating dead time.

### v12 — live progress bar
Replaced fixed checkpoints with a background job + a loop that ticked the bar
up once per second while the job was alive, snapping to the true phase
boundary only when the tool actually exited. Correct in isolation, but
progress bars are fundamentally the wrong abstraction for tools with unknown
completion time — see v14.

**Bug found:** HTTrack exited instantly (status 255) with an empty output
directory. Root cause: the `--continue` flag tells HTTrack to resume a
previous mirror from its cache. On a brand-new output directory with no
`hts-cache` yet, there's nothing to resume, and in a non-interactive/
backgrounded context HTTrack hard-fails instead of falling back to a fresh
mirror. **Fix:** only pass `--continue` if `hts-cache` already exists for that
target; otherwise omit it. Confirmed against a real manual `httrack` run
(exit 0, files written) vs. the scripted run (exit 255, empty dir) using the
exact same flags minus `--continue`.

### v13 — reachability & dependency checks
Added upfront checks: all three tools (`feroxbuster`, `katana`, `httrack`)
present on `PATH`, and a `curl -sI` reachability probe before starting any
phase — fail fast and loud instead of three phases deep with nothing to show
for it.

### v14 — hard time budget instead of a progress bar
Replaced the percent-bar entirely. A progress bar implies a *known* end
point; an open-ended crawl doesn't have one, so any percentage it shows is a
guess dressed up as data. Switched to `-T1`..`-T5` flags mapping to a total
time budget (2/5/10/60/120 min), split evenly across the 3 phases, with a
live `MM:SS` countdown per phase instead of a percentage.

Each phase now runs the tool in the background, counts down its slice of the
budget, and — if the tool hasn't finished when the slice runs out — sends
`SIGINT` (same signal as a manual Ctrl+C, so tools flush partial results
instead of losing everything), followed by `SIGKILL` after a 5s grace period
if it's still alive. Verified with synthetic tests: a job that finishes early
doesn't wait out the rest of its budget, and a job that never finishes gets
cut off at the exact boundary.

**Known artifact:** Feroxbuster reliably exits via `SIGKILL` (status 137)
rather than `SIGINT` (130) when time-limited, because its Ctrl+C handling is
built around an interactive terminal menu that doesn't exist when the process
is backgrounded with no tty. This is expected, not a failure — partial
results are already on disk regardless, since Feroxbuster streams matches to
its output file as it finds them. The script's status-code check was updated
to not flag 137 as an error.

### v15 — dynamic scheme detection
Earlier versions hardcoded a `http://` fallback when no scheme was given.
This happened to work against the HTB lab target used for testing (HTTP-only)
but would silently break on the far more common case of an HTTPS-only or
HTTP→HTTPS-redirecting real-world site. Replaced with actual detection: try
`https://` first, fall back to `http://`, fail clearly if neither is
reachable. Also added a wordlist-existence check (checks a few common
seclists/dirb install paths) so a missing wordlist fails loudly instead of
Feroxbuster dying silently mid-phase.

**Validated against two different targets** (one HTTPS real-world domain, one
HTTP-only lab target) to confirm the pipeline is actually generic and not
tuned to a single site. Both produced populated `full-mirror/` output and a
working Bearer scan.

### Renamed: site-spider (formerly sourcepuller)

---

## Research: what comes after site-spider

Before building further phases, did a pass on what open-source tooling
already exists for the later pipeline stages, to avoid rebuilding things that
are already mature:

- **Bearer's language coverage does not include PHP** — a real gap, since the
  test target's PHP source (`index.php`) was silently skipped during
  scanning, and only the 12 JS files were ever analyzed. This needs to be
  addressed before the SAST phase can be considered reliable on typical
  targets.
- **Semgrep** identified as the most likely PHP-coverage fix — broadest
  open-source language coverage, same `file:line` output contract as Bearer,
  community OWASP rulesets available.
- **Nuclei** identified as the natural DAST addition — same ecosystem as
  Katana (ProjectDiscovery), accepts URL lists directly (i.e. can consume
  `urls/all.txt` from site-spider with no glue code), template-based so
  findings come with a specific ID and evidence rather than a vague signature.
- Manual Nikto run against the lab target surfaced a plausible LFI pattern
  (`index.php?page=...` path traversal) — useful concrete example of the kind
  of DAST-only finding that needs correlating against the mirrored source to
  become actionable.
- **Correlation/normalization layer**: rather than hand-roll deduplication
  and cross-referencing across tools, evaluated existing open-source options.
  DefectDojo (OWASP Flagship project, 200+ tool parsers, built-in dedup
  engine) and Faraday (positioned more toward offensive-security/pentest
  workflows specifically) are the two candidates; Faraday looks like the
  better fit given this project's use case.
- **AI reasoning layer**: intended to sit on top of the correlated findings,
  not raw tool output — the core design constraint being that the AI should
  only ever receive narrow, structured, pre-correlated context (specific
  finding + specific code snippet), never a full raw dump of any single
  tool's output. This is the direct fix for the "AI gets confused by a full
  linpeas/tool dump" problem observed in earlier manual testing.

Next planned work: wiring Semgrep and Nuclei in as additional phases, and a
first-pass correlation script that matches DAST parameter names against
grep'd source locations.
