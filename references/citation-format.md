# Citation Entry Format

## Standard Entry

```markdown
**[5]** LastName, FirstName; LastName, FirstName. "Article Title."
*Publication Name*, vol. X, article/issue Y, YEAR.
DOI: 10.xxxx/xxxxx.
<https://full-url-here>
Brief description of specific data extracted (with units, sample sizes).
```

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
