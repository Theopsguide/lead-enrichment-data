# Lead Enrichment Data

Secure bridge for lead enrichment between Claude Code cloud (sandboxed) and local HQ workspace.

## Architecture

This repo serves as the data transfer layer between:
- **Cloud Claude Code** (claude.ai/code) - runs web research in isolated sandbox
- **Local HQ Workspace** - has MCP access to Google Sheets

```
Cloud VM (sandboxed) ──► GitHub (this repo) ──► Local HQ (MCP enabled)
```

## Workflow

### 1. Export (Local HQ)
```
/leads export
```
Exports unenriched leads from Google Sheets to `data/leads-pending.json`

### 2. Enrich (Cloud claude.ai/code)
```
# Open https://claude.ai/code
# Clone this repo
/leads enrich
```
Runs web research in sandboxed environment, creates PR with results

### 3. Merge (Local HQ)
```
/leads merge
```
Pulls PR, runs security scan, imports clean data to Google Sheets

## Security

- Web scraping happens in isolated cloud VM (gVisor sandbox)
- Results pass through security scan before entering trusted environment
- Suspicious entries are flagged and quarantined

## Files

| File | Purpose |
|------|---------|
| `data/leads-pending.json` | Input: leads to be enriched |
| `data/leads-enriched.json` | Output: enriched results |
| `schemas/enriched-lead.json` | JSON schema for validation |
| `.claude/commands/leads.md` | Cloud enrichment command |
