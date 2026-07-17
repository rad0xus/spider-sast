# AGENT.md - The Vision for crawl-hunt

## Project Overview

`crawl-hunt` is an ambitious, comprehensive open-source CLI tool designed to automate the entire process of web application security assessment and vulnerability hunting. It combines aggressive crawling, extensive enumeration, source code mirroring, multi-language SAST/DAST scanning, correlation of findings, and AI-powered analysis into a single, cohesive pipeline that runs entirely from the command line. The ultimate goal is to empower security researchers with a "one-command-to-rule-them-all" experience that mimics what a highly skilled, tool-augmented pentester would do — but at machine scale and with unparalleled breadth.

Everything happens through the CLI: no manual switching between tools, no fragmented outputs, no need for complex setups beyond initial dependencies. The tool fetches, prepares, scans, correlates, and even provides step-by-step exploitation guidance.

## Why No Such Open-Source Tool Exists Today (The Gap)

Despite the abundance of excellent individual open-source security tools, there is currently **no single open-source project** that holistically addresses the full spectrum of modern web security reconnaissance and analysis in one unified pipeline. Here's why this vacuum exists and why `crawl-hunt` fills it:

1. **The Enumeration Problem**: Effective enumeration and content discovery *always* requires multiple tools. Tools like `gau`, `waybackurls`, `ffuf`, `dirsearch`, `gobuster`, `hakrawler`, `katana`, etc., each excel in specific niches (historical archives, brute-forcing, JavaScript endpoint extraction, etc.) but have blind spots. One tool might miss client-side rendered paths; another might choke on authentication or rate limits. No project systematically chains them with success-based filtering and deduplication at scale.

2. **Source Code Preparation Challenges**: Mirroring a full website (including dynamic content, JS bundles, API responses) while preserving structure for static analysis is non-trivial. Tools like `wget`, `httrack`, or `Browsershot` exist, but integrating them seamlessly with live crawling, handling JavaScript-heavy SPAs, respecting robots.txt ethically, and preparing a clean source tree for SAST scanners requires custom orchestration that's missing from existing frameworks.

3. **Language and Technology Agnostic Scanning**: Modern web apps use a dizzying array of languages and frameworks: JavaScript (Node, React, Vue), PHP, Python, Java, .NET, Ruby, Go, and config files (.env, web.config, .htaccess, etc.). Individual SAST tools (Semgrep, Trivy, Bandit, etc.) and DAST tools (ZAP, Nuclei, Nikto) cover subsets. No pipeline runs the *right combination* conditionally, correlates findings across them, or normalizes outputs into a unified format.

4. **Lack of AI Integration in CLI Pipelines**: Post-scan analysis typically requires human review of raw logs. There are no mature open-source CLI tools that feed the entire crawled source + findings into local or API-based AI agents for deep reasoning, vulnerability chaining, and precise exploitation walkthroughs.

5. **Fragmentation and UX Hell**: Researchers juggle 20+ tools, custom scripts, Docker containers, and note-taking apps. Outputs are unstructured text dumps. Correlation is manual. Time-boxing, logging, and reproducibility are afterthoughts.

`crawl-hunt` eliminates this fragmentation by enforcing a strict philosophy: **multi-tool coverage**, **structured data**, **fail-fast checks**, **time bounds**, and **target-agnostic design**.

## Core Vision and Pipeline Phases

The pipeline is modular, extensible, and designed for continuous evolution. Each phase builds on the previous:

### 1. Target Input and Initial Recon
- Accept single URL, list of URLs, or scope file.
- Passive recon: DNS, WHOIS, certificates, subdomains (via Amass, Subfinder, etc.).
- Active crawling with multiple engines (Katana, Hakrawler, gau+wayback, etc.) — always combining outputs.

### 2. Extensive Enumeration & Content Discovery
- Multi-tool URL discovery, parameter fuzzing, directory brute-forcing.
- JS endpoint extraction, API harvesting, form/endpoint mapping.
- **Rule of Enum**: Chain 5+ tools, deduplicate, filter live endpoints only.
- Output: Comprehensive `urls/` directory with categorized lists (all, js, api, admin, etc.).

### 3. Website Source Mirroring & Preparation
- Full recursive mirror of live endpoints using optimized tools.
- Handle JS rendering (headless browser integration if needed).
- Extract and organize: HTML, JS bundles (unminify where possible), CSS, configs, backend files if exposed.
- Prepare clean source tree under `full-mirror/` for SAST.
- Ethical considerations: respect robots.txt, rate limiting, scope.

### 4. Multi-Language Scanning (SAST + DAST + Misc)
- **SAST**: Run language-specific and polyglot scanners (Semgrep rules for 30+ languages, Trivy, Grype, etc.) on the mirror.
- **DAST**: Nuclei templates, ZAP active scan (time-bounded), custom scripts.
- **Specialized**: Secret scanning (gitleaks, trufflehog), dependency checks, config audits.
- Support for JS, PHP, Python, Java, .config files, etc. — detect presence automatically.
- Every scanner produces **structured output**: `{tool, finding, location, evidence, severity}`.
- Logs and raw outputs preserved for debugging.

### 5. Correlation Engine
- Cross-reference DAST findings (e.g., XSS param) with SAST source locations.
- Chain vulnerabilities (e.g., IDOR + weak auth).
- Deduplicate and prioritize high-impact issues.

### 6. AI Agent Layer (The "Agent" in AGENT.md)
- Feed the entire dataset (mirrored source, all findings, URL graphs) into one or more AI agents.
- Agents run locally (via Ollama/LM Studio) or via API with strict prompting.
- Capabilities:
  - Summarize attack surface.
  - Reason about exploitability.
  - Generate precise exploitation playbooks.
- **CLI-Driven Exploitation Guidance**: For each high-confidence vuln, the AI outputs step-by-step instructions like:
  ```
  To exploit the reflected XSS at /search?q=:
  1. Open browser to: https://target.com/search?q=<script>alert(1)</script>
  2. Observe popup.
  3. For advanced: Use this payload: ...
  4. Check affected endpoints: ...
  ```
- Agents can suggest browser routes, Burp/Repeater setups, or PoC code.

### 7. Reporting and Export
- Unified HTML/JSON/Markdown reports.
- Interactive CLI summary with colors and priorities.

## Technical Principles
- **CLI-First**: `crawl-hunt scan https://example.com --depth aggressive --ai-enabled`
- **Extensible Phases**: New tools added as scripts following templates.
- **Reproducible**: Full logging, version pinning of tools.
- **Ethical by Default**: Time bounds, scope enforcement, disclaimers.
- **Performance**: Parallel where safe, timeouts everywhere.

## AI Agents in Detail
The AI component is transformative:
- Multiple specialized agents: Recon Agent, Code Analyst, Exploit Crafter, Report Writer.
- Prompt chaining and tool-use (e.g., agents can invoke sub-scans).
- Everything surfaced via CLI — no leaving the terminal.
- Privacy-focused: Prefer local models.

## Disclaimer and Usage Policy
**This tool is provided solely for security research, authorized penetration testing, and bug bounty programs on targets you own or have explicit written permission to test.**

- **Do not use on unauthorized systems.** Unauthorized access or scanning violates laws (e.g., CFAA in the US).
- All activities should comply with applicable laws and target terms of service.
- The maintainers bear no responsibility for misuse. Use responsibly to improve security, not harm it.
- Contributions that enable malicious use will not be accepted.

By using `crawl-hunt`, you agree to ethical hacking principles. Happy (and legal) hunting!

## Call to Action
Join the vision! Add scanners, improve AI prompts, enhance correlation, or bring your unique vibe — as long as it pushes the comprehensive, multi-tool, AI-augmented security pipeline forward.

See CONTRIBUTING.md for details.