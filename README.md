# crawl-hunt

An automated security pipeline that enumerates a target web application, mirrors discoverable source pages and assets, and runs static application security testing (SAST) to identify code-level vulnerabilities, software bugs, and potential CVEs.

## Overview

Scanning web applications often creates an overwhelming amount of raw output across multiple tools without clear correlation between endpoints and source code. **crawl-hunt** addresses this by bridging content discovery, local asset mirroring, and static code analysis into a streamlined workflow to pinpoint actionable security flaws.

## Pipeline Architecture

1. **Enumeration & Discovery**
Discovers accessible paths, hidden directories, and endpoints using complementary crawlers (e.g., Feroxbuster, Katana).
2. **Source & Asset Mirroring**
Downloads client-accessible source files, JavaScript bundles, and page assets (e.g., via HTTrack or targeted pulls) strictly for endpoints that return valid responses (2xx/3xx).
3. **Static Analysis (SAST)**
Executes static security scanners (e.g., Semgrep, Bearer) over the mirrored codebase to flag vulnerable functions, insecure configurations, and potential CVE matches tied to specific files and line numbers.

## Current Status

The primary active module is **`site-spider`**, which handles the initial 3-phase enumeration and mirroring pipeline:

| Phase | Tool | Function |
| --- | --- | --- |
| 1 | [Feroxbuster](https://github.com/epi052/feroxbuster) | Directory and content discovery |
| 2 | [Katana](https://github.com/projectdiscovery/katana) | JavaScript-aware web crawling |
| 3 | [HTTrack](https://www.httrack.com/) | Full source and asset mirroring of discovered paths |

### Key Features

* **Protocol Auto-Detection:** Determines whether to use `https://` or `http://` for bare hostnames.
* **Time Budget Controls:** Enforces configurable execution limits (`-T1` through `-T5`) split across pipeline phases with graceful process termination (`SIGINT`) to preserve partial output.
* **Dependency Validation:** Verifies tool binaries and wordlists before launch to prevent late-stage silent failures.

## Usage

```bash
chmod +x site-spider
./site-spider <target> [-T1|-T2|-T3|-T4|-T5]

```

## Planned Enhancements

* **Broader SAST Language Coverage:** Integration of Semgrep and PHP-focused analyzers (e.g., Progpilot) to support server-side script analysis.
* **CVE & Vulnerability Mapping:** Cross-referencing identified code patterns and third-party dependencies against known CVE databases.
* **Output Normalization:** Consolidating scanner outputs into a unified structured report (`{tool, finding, file, line, cve_id}`).
