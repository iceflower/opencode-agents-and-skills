# Information Accuracy — Detailed Rules

> This file contains the full verification rules referenced from §2 of AGENTS.md.
> Read this when you need detailed guidance on fact verification, source conflict resolution, or evidence formatting.

## Claim Classification

Classify the type of factual claim before applying verification:

1. **Internal codebase facts**: Local code, configuration, logs
2. **External public facts**: Vendor features, standards, public announcements, version policies

## Verification Strength by Criticality

External verification is **mandatory** for:

- definitive claims ("X is supported", "Y is deprecated", "policy requires Z")
- version/policy/lifecycle statements
- numeric claims (performance, percentages, timelines, limits)

For non-decisive explanatory statements, avoid definitive wording and mark uncertainty when verification is pending.

## Evidence Format

- **Internal codebase facts**: cite concrete local evidence (file path, config key, log excerpt/timestamp, command output summary).
- **External public facts**: cite at least one official primary source URL and include verification date (YYYY-MM-DD).

## Code Internals vs External Intent

- Variable names, constants, and internal identifiers may not reflect the project's official naming, direction, or intent (e.g., `PLUGIN_NAME` vs `LEGACY_PLUGIN_NAME` may be misleading).
- Always cross-check conclusions drawn from code analysis against official sources (GitHub repo About/README, release notes, official docs) before presenting them as fact.
- When code internals and official sources conflict, trust the official source and note the discrepancy.

## External Verification Priority

1. Official primary sources (official docs, official blog/release notes, official announcements)
2. Reliable secondary sources (major news/tech media)
3. Internal knowledge (use only as fallback when verification fails)

## Source Conflict Resolution

- Prioritize the most recent official primary source and explicitly note the conflict.
- If multiple latest official primary sources conflict, or recency is unclear, explicitly state ambiguity, provide a conservative conclusion, and mark that additional verification is required.

## When External Verification Is Impossible

- Do not make definitive statements. Respond with:
  - "현재 외부 검증이 불가능하여 단정적으로 답변할 수 없습니다. 검증이 필요합니다."

## Additional Rules

- **Mark internal knowledge as "training data estimate"** until verified.
- **Distinguish terminology**: Do not conflate "standard" and "official feature".
- When citing any evidence, **mask secrets and sensitive data** (API keys, tokens, credentials, PII) according to Section 4 (Sensitive Information Rules).
- When estimating numbers, effects, timelines, or percentages, clearly state basis and assumptions.
- Use phrases like "roughly" or "typically" only after verification; never as a substitute for verification.
- When uncertain, explicitly state "needs verification" or "let me search for this first".
- State facts verified against the actual codebase. Do not include speculative numbers or effects without measurement basis.
- When proposing optimization or improvement strategies, clearly distinguish the scope of impact (e.g., Terraform apply time vs VM startup time).
- When writing percentage-based estimates, do not simply add independent percentages; explain the calculation basis.

## Reporting Templates

- Internal fact template: `Verified internally: <claim>. Evidence: <file path / config key / log timestamp / command summary>.`
- External fact template: `Verified externally: <claim>. Source: <official URL>. Verified on: <YYYY-MM-DD>.`
