# V4.1: UniversalDropZone Enhancements

> Coder-3 stub | D164b | Source: D161 spec 05 finding — drop zone already shipped

## What

D161 discovered UniversalDropZone (985 LOC) already handles D85's file-drop requirements. V4.1 improvements:

1. **PDF disambiguation via Vision** — PDFs without contract/invoice keywords currently fall to `ASK` (opens chat asking what to do). Use first-page Vision API inspection to auto-classify.
2. **Multi-file drop** — currently takes `files[0]` only (line 375). Support batch processing with progress indicator.
3. **Dispute evidence category** — add `DISPUTE_EVIDENCE` file type for PDFs/images related to billing disputes.
4. **Timesheet category** — add `TIMESHEET` for VA time tracker CSV imports.

## Scope per enhancement

| Enhancement | Files | ~LOC |
|---|---|---|
| PDF Vision classification | UniversalDropZone.tsx (detectFileCategory), capture API (add classify-pdf endpoint or reuse capture) | ~40 |
| Multi-file drop | UniversalDropZone.tsx (batch processing loop + progress UI) | ~60 |
| Dispute evidence category | UniversalDropZone.tsx (new category + handler), disputes API (accept evidence attachment) | ~30 |
| Timesheet category | UniversalDropZone.tsx (new category + CSV header detection), time-tracker API | ~25 |

## Total rough LOC

~155 LOC across 2-4 files per enhancement.

## Dependencies

- None of these block V4.0
- PDF Vision classification adds an API call per dropped PDF — cost consideration
- Multi-file needs design decision: process in parallel or sequentially? Sequential with progress bar is simpler.
