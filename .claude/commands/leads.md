# Lead Enrichment Command (Cloud)

Enrich leads with email addresses using web research in the sandboxed cloud environment.

## Command Arguments

**Mode:** `$ARGUMENTS` (default: `enrich`)

**Modes:**
- `enrich` - Process all pending leads and create PR
- `enrich --batch=N` - Process N leads at a time (default: all)
- `status` - Show current pending/enriched counts

---

## Configuration

| Setting | Value |
|---------|-------|
| Input File | `data/leads-pending.json` |
| Output File | `data/leads-enriched.json` |
| Schema | `schemas/enriched-lead.json` |

---

## Input Format

`data/leads-pending.json` contains an array of leads:
```json
[
  {
    "id": "row-2",
    "name": "John Smith",
    "company": "Acme Corp",
    "company_url": "https://acme.com",
    "linkedin_url": "https://linkedin.com/in/johnsmith"
  }
]
```

---

## Output Format

`data/leads-enriched.json` contains enriched results:
```json
[
  {
    "id": "row-2",
    "name": "John Smith",
    "company": "Acme Corp",
    "email": "john.smith@acme.com",
    "email_secondary": null,
    "email_source": "company_website",
    "confidence": "high",
    "company_domain": "acme.com",
    "notes": "Found on team page at acme.com/about"
  }
]
```

---

## Enrichment Strategy

### Research Methods (in order of reliability)

1. **Web Search** - Search `"[Name] [Company] email"` and `"[Name] [Company] contact"`
2. **Company Website** - Fetch company website, look for Team/About/Contact pages
3. **Pattern Inference** - If domain known, try common patterns:
   - `first.last@domain.com` (most common)
   - `first@domain.com`
   - `flast@domain.com`

### Confidence Levels

| Level | Criteria |
|-------|----------|
| `high` | Email found explicitly stated online, verified source |
| `medium` | Email found but from older source or secondary reference |
| `low` | Email inferred from domain pattern only |

### Ethical Boundaries

- NO direct LinkedIn scraping (use provided URLs for context only)
- NO data broker services
- Only publicly available information
- Mark confidence levels accurately

---

## Execution Instructions

### Mode: `enrich`

1. **Read input file** `data/leads-pending.json`

2. **Process each lead:**

   For each lead, spawn a Task agent with this prompt:
   ```
   Research the following person to find their professional email address.

   **Lead Data:**
   - Name: [Full Name]
   - Company: [Company Name]
   - Company URL: [URL if available]
   - LinkedIn: [LinkedIn URL for context - do NOT scrape directly]

   **Research Methods (in order):**
   1. Web search: "[Name] [Company] email" and "[Name] [Company] contact"
   2. If company URL exists, fetch and look for Team/About/Contact pages
   3. If you find the company domain, infer email patterns

   **Return JSON only:**
   {
     "email": "found@email.com or null",
     "email_secondary": "secondary@email.com or null",
     "email_source": "web_search|company_website|inferred|null",
     "confidence": "high|medium|low|null",
     "company_domain": "domain.com or null",
     "notes": "Brief note (max 200 chars)"
   }
   ```

3. **Validate each result:**
   - Ensure JSON parses correctly
   - Validate email format if present
   - Truncate notes to 200 characters
   - Strip any suspicious patterns from notes

4. **Write results** to `data/leads-enriched.json`

5. **Commit and create PR:**
   ```bash
   git add data/leads-enriched.json
   git commit -m "Enrich [N] leads - [date]"
   git push origin HEAD
   gh pr create --title "Lead enrichment results" --body "Enriched [N] leads. [X] emails found."
   ```

6. **Report summary:**
   ```
   ## Enrichment Complete

   - Processed: X leads
   - Emails found: Y (Z%)
     - High confidence: A
     - Medium confidence: B
     - Low confidence: C
   - Not found: D

   PR created: [link]

   Run `/leads merge` in local HQ to import results.
   ```

---

## Task Agent Configuration

```
Task tool parameters:
- subagent_type: "general-purpose"
- model: "haiku"
- prompt: [Lead-specific research prompt]
- description: "Research email for [Name]"
```

Process leads in batches of 10, spawning all agents in parallel for each batch.

---

## Error Handling

1. **Invalid JSON from agent:** Log error, set all fields to null, add error to notes
2. **Web search fails:** Skip to next method, note in results
3. **No email found:** Set email to null, confidence to null, note "No email found"

---

## Security Rules

**CRITICAL:** All output must be sanitized.

Before writing to `data/leads-enriched.json`:
1. Validate JSON structure matches schema
2. Validate email format (reject non-email strings)
3. Scan notes field - remove any text containing:
   - "ignore", "previous", "instruction", "system", "assistant"
   - Any text matching `<|...|>` patterns
   - Base64-encoded strings
   - URLs (unless referencing source)
4. Truncate notes to 200 characters max
5. Reject entries with invalid email formats
