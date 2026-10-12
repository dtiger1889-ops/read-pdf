---
name: read-pdf
description: Extract and read PDF text, especially browser print-to-PDF articles and PDF drop folders. Use when content matters; for forms, signatures, or layout inspection, use a workflow that renders the relevant pages.
---

# read-pdf -- token-efficient PDF text extraction

Extract text locally before loading a PDF into the model. Render specific pages only
when the text layer is missing or extraction order obscures the content.

## Extraction choices

- Prefer `pdftotext` to file-reading tools that render every page as an image.
- Avoid full annotation dumps: browser prints may contain many duplicate navigation
  links. Inspect only the metadata or links the task needs.
- Use a deterministic Markdown converter if text extraction loses structure the task
  needs; ordinary prose rarely needs conversion.

## Procedure

1. **Triage (optional, for big or multi-file batches)** -- page count + producer:
   ```
   pdfinfo "file.pdf" | grep -iE "pages|producer|creator"
   ```
2. **Extract** to a scratch file (never inline for unknown sizes):
   ```
   pdftotext -q "file.pdf" scratch.txt && wc -c scratch.txt
   ```
3. **Read by size:** for manageable text, read the whole .txt. For larger files,
   search relevant sections or read in chunks; continue to EOF when the task requires
   the whole document. Blank pages need no extra processing.
4. **Page ranges** when only part matters: `pdftotext -f 2 -l 5 ...`

Requires Poppler (`pdftotext`, `pdfinfo`). Check the current installation first. On
Windows, use an installed Poppler release or its full executable path; do not assume
Git for Windows includes it. The commands above use a Bash shell; in PowerShell,
use `Select-String` and `(Get-Item scratch.txt).Length` for filtering and byte counts.

## Expected artifacts (not corruption -- read through them)

| Producer (from pdfinfo) | Source | Artifacts |
|---|---|---|
| cairo (Mozilla Firefox) | Firefox / Firefox mobile | May repeat title, source URL, page number and timestamp in page headers. Some Reddit prints omit usernames; do not attribute comments when the PDF lacks them. |
| Skia/PDF (Chrome UA) | Brave / Chrome mobile | May interleave overlay or navigation text with body lines. Continue if meaning is clear; inspect the affected page if it is not. |

## Reddit prints specifically

- `Join the conversation` may mark the comments section. Read the comments when
  relevant to the user's request.
- **Collapsed comments are simply absent from the PDF.** No extraction fixes
  that; expansion has to happen in the browser before printing. If a big
  thread's text ends right after `Join the conversation`, say so -- don't guess
  at comments that aren't there.
- Skip suggested posts, navigation and "Related Answers" when they are unrelated.
- Duplicate prints happen (same article printed twice under different names).
  On multi-file batches, compare the first ~200 chars before reading both.

## Fallbacks (only if pdftotext output is near-empty for pages that visibly have content)

Very little text on a visibly full article page suggests a raster page or a failed text
layer. Inspect that page to confirm, then escalate in order:
1. A text-oriented PDF MCP tool with explicit extraction status, if available.
2. OCR (tesseract) or page rendering on the SPECIFIC pages only -- image
   tokens, last resort.

## Links (on request only -- pdftotext drops them)

Firefox prints: the source URL is in the extracted page header, no extra work.
For in-body links, use pypdf: collect unique `/A` -> `/URI` values across pages,
then filter obvious site chrome (nav, login, notifications, recommended-post
sidebars) before showing.

## After reading

When processing a drop/inbox folder includes filing, move the original PDF to its
long-term reference folder or dated archive after reading it. A read-only request
does not authorize moving it. Keep the PDF as the source; the scratch .txt is disposable.
