# crawl-hunt

**An open-source pipeline that crawls a web target, pulls down as much of its
source code as it can, and runs it through vulnerability-scanning tools — with
the long-term goal of an AI layer that turns raw scanner noise into a precise,
human-readable "here's exactly how to exploit this" report.**

## Why this exists

If you've ever run a handful of recon/scanning tools against a target — Nikto,
Feroxbuster, Katana, a SAST scanner, linpeas on a box you've already landed on
— you know the real bottleneck isn't finding signal, it's **finding the signal
inside the noise**. Each tool speaks its own format, none of them know about
each other's findings, and a raw dump of any single tool's output is usually
too long and unstructured for even an experienced person (or an LLM handed the
whole file) to reliably act on. A DAST tool might flag a suspicious parameter;
a SAST tool might flag a dangerous function; almost nothing available for free
today automatically connects those two dots and shows you the specific line
of code behind the specific exploitable endpoint.

That gap — turning "20 tools' worth of loosely related output" into "here is
the one thing to try, and here's why" — is what crawl-hunt is trying to close,
starting with the web-application slice of that problem (not post-exploitation
/ privesc parsing, at least not yet).

This is explicitly a **community tool**. Nobody has deep expertise in every
open-source scanner out there, so the intent is to keep the pipeline modular
enough that anyone can drop in support for a new tool via pull request. See
[CONTRIBUTING.md](CONTRIBUTING.md).

## Current status

The only working piece today is **`site-spider`**, a 3-phase recon/mirroring
pipeline:

| Phase | Tool | Purpose |
|---|---|---|
| 1 | [Feroxbuster](https://github.com/epi052/feroxbuster) | content/directory discovery |
| 2 | [Katana](https://github.com/projectdiscovery/katana) | JS-aware crawling |
| 3 | [HTTrack](https://www.httrack.com/) | full source mirror of what was found |

Features as of now:
- Auto-detects `https://` vs `http://` for a bare hostname instead of assuming one
- Hard time budget via `-T1`..`-T5` (2/5/10/60/120 min total, split evenly across
  the 3 phases) with a live countdown — each phase is cut off gracefully
  (`SIGINT`, so tools flush partial results) rather than run indefinitely
- Dependency and wordlist path checks up front, so failures are loud and
  immediate instead of silent
- Works against any target, not one specific site — validated against both an
  HTTPS real-world domain and an HTTP-only lab target

Usage:
```
chmod +x site-spider
sourcepuller-style usage:
  site-spider <target> [-T1|-T2|-T3|-T4|-T5]
```

See [worklog.md](worklog.md) for the detailed history of how it got here,
including bugs found and fixed.

## Where this is going

The full intended pipeline, in order:

1. **Enumeration** — thorough recon before anything else. Rule of thumb: never
   rely on a single tool for enumeration, since each one has blind spots.
   Feroxbuster + Katana today; Nikto and/or Nuclei planned as additional
   enumeration passes (Nikto's header/config/known-vuln checks and Nuclei's
   template-based checks catch different things than a pure content-discovery
   crawler does).
2. **Source acquisition** — pull down source only from endpoints that actually
   returned something (2xx/3xx), not from paths that 404'd. HTTrack today;
   may be supplemented with targeted `curl` pulls for endpoints the mirror
   missed.
3. **Static analysis (SAST)** — [Bearer](https://github.com/Bearer/bearer) is
   the current tool, but **it does not support PHP** (its language coverage is
   JS/TS, Ruby, Java, Go, and Python) — a real gap, since PHP is extremely
   common on the kind of targets this tool is meant for. Planned additions:
   [Semgrep](https://semgrep.dev/) (broadest language coverage of any open
   SAST tool, PHP included, same `file:line` output contract as Bearer) and/or
   PHP-specific tools like Progpilot or PHPStan with security rules.
4. **Dynamic analysis (DAST)** — running scanners against the *live* target,
   not just its source. Nikto already found a plausible LFI pattern in manual
   testing; [Nuclei](https://github.com/projectdiscovery/nuclei) is the
   planned addition here since it shares an ecosystem with Katana and accepts
   URL lists directly from the enumeration phase.
5. **Correlation layer** — normalizing and cross-referencing findings from all
   of the above into one structured record (`{tool, finding, file, line,
   endpoint, evidence}`) instead of a pile of separate text/log files. Rather
   than build this from scratch, the plan is to lean on existing open-source
   tooling built exactly for this (e.g.
   [DefectDojo](https://github.com/DefectDojo/django-DefectDojo) or
   [Faraday](https://github.com/infobyte/faraday)) rather than reinvent
   finding-deduplication.
6. **AI reasoning layer** — once findings are structured and correlated, the
   plan is to point an agent (currently looking at
   [Antigravity](https://antigravity.google/)) at the full saved result set —
   mirrored source, enumeration output, SAST/DAST findings — and have it
   produce a precise, step-by-step "this is exploitable, here's exactly how"
   report. The key design constraint here: the agent should only ever be
   handed *narrow, structured, pre-correlated* context, never a raw dump of
   every tool's full output — that's what makes the difference between a
   useful answer and a confused one.

## Contributing

Contributions are very welcome, especially:
- Wiring in new enumeration/SAST/DAST tools as additional pipeline phases
- Improving correlation logic between DAST findings and source-code locations
- Anything that makes the eventual AI-layer input more structured and precise

See [CONTRIBUTING.md](CONTRIBUTING.md) for how the pipeline is structured and
how to submit a new tool integration.
# crawl-hunt
