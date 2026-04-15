# Citation Entry Format

## Standard Entry

```markdown
**[5]** LastName, FirstName; LastName, FirstName. "Article Title."
*Publication Name*, vol. X, article/issue Y, YEAR.
DOI: 10.xxxx/xxxxx.
<https://full-url-here>
**Tier:** 1
Brief description of specific data extracted (with units, sample sizes).
```

### Source Quality Tiers

- **Tier 1:** Peer-reviewed papers, government/institutional reports
- **Tier 2:** Manufacturer specs, established reference sites, university publications
- **Tier 3:** Industry blogs, conference talks, well-known practitioners
- **Tier 4:** Forums, personal blogs, GitHub discussions, social media

The tier helps readers and audit agents quickly assess source reliability.
When sources conflict, higher-tier sources take precedence by default.

## Entry with Quality Flag

```markdown
**[2]** "Page Title." *Site Name*, n.d.
<https://full-url-here>
Data extracted: [specific details].
Note: [quality concern — e.g., "article shows signs of AI-generated content;
figures are plausible but not confirmed against primary sources."]
```

## Entry with Access Failure

```markdown
**[9]** LastName, FirstName. "Article Title." *Journal*, YEAR.
DOI: 10.xxxx/xxxxx.
<https://original-url>
**Access:** PDF extraction failed; data sourced from [secondary source N].
```

## Entry Marked as Retracted

```markdown
**[12]** LastName, FirstName. "Article Title." *Journal*, YEAR.
<https://url>
**Status:** RETRACTED — claim not found in source upon audit verification.
Originally cited for: [what it was supposed to support].
```

## Retraction Ledger Format

When the update workflow removes claims due to lost citation support, record
each removal in `retraction-log.md` in the topic directory. Each entry
documents what was removed and why, providing an audit trail beyond git
history.

```markdown
## <ISO-8601 timestamp>

- **Removed claim:** "<exact text of the removed claim>"
- **Failed citation:** [N] Author, Year, <URL>
- **Reason:** <one of: URL returns 404 | Content drift — originally cited
  data no longer present | Source explicitly retracted | User confirmed
  material drift>
- **Discovery pass:** <whether the update's discovery agents found
  replacement sources — yes/no>
```

If no claims are removed during an update run, do not create or modify the
retraction ledger.

## Rules

1. **Number sequentially.** Do not skip numbers.
2. **Include author, title, publication, DOI** when available.
3. **Note the specific data extracted** — not just "useful article."
4. **Flag source quality concerns** (AI-generated content, testimonial-only,
   secondary citation). The audit agent checks these flags for honesty.
5. **Mark retracted sources** with `**Status:** RETRACTED` rather than deleting.
   This keeps downstream citation numbers stable across all files.
6. **Record access status** when a source could not be directly fetched.
   Always note the fallback path used to obtain the data.
