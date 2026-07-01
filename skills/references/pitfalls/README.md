# Case-Specific Pitfalls

This directory contains post-mortem notes from specific DeFi incident investigations. Each file documents what went wrong during the investigation, how it was detected, and the fix applied.

## When to Add a File

After completing an investigation using the `defi-incident-investigation` skill, add a pitfalls file if you encountered any of the following:

- A verification trap not already covered in `references/pitfalls/general.md`
- A case-specific edge case that could mislead future investigations of similar attack patterns
- A new hallucination pattern or data source quirk
- A schema compliance issue that required a new vocabulary entry

## File Naming

`<ProtocolName>-<YYYYMMDD>.md` — use the same protocol name and date as the incident report.

Example: `Taiko-20260622.md` for an incident with id `Taiko-20260622`.

## Template

```markdown
# <ProtocolName>-<YYYYMMDD> Case Pitfalls

## Case Summary
- **Incident:** <one-line description>
- **Loss:** <amount>
- **Attack pattern:** <attack type>

## Pitfalls Encountered

### 1. <Pitfall Title>

**What happened:** <description of what went wrong during the investigation>

**How detected:** <how the issue was discovered — which verification step caught it>

**Fix:** <what was done to fix the issue>

**Lesson:** <generic takeaway applicable to future investigations>

### 2. <Pitfall Title>
...
```

## Guidelines

- Focus on what went wrong during the INVESTIGATION, not the attack itself (the attack details belong in the report JSON)
- Generalize the lesson — don't just describe the case, extract the reusable insight
- If the pitfall is broadly applicable (not case-specific), consider adding it to `references/pitfalls/general.md` instead
- Do NOT include real API keys, user-specific file paths, or environment-specific details
