# Module 3 — My Rules for Importing Books

MSAI-5523: Language of Computation

**Status: FINAL and APPROVED business rules.**

## What I want this program to do

I want a Python program that reads each book in a CSV file, checks the ISBN values using the same rules every time, and imports trustworthy book information into books.db. I do not want the program to use AI, contact the internet, or guess missing digits when it runs. The original books.csv and books-batch2.csv must never be edited. I want to keep useful book records even when their ISBNs are missing or bad, and I want a clear paper trail of what happened.

For validation, ISBN-10 must contain nine digits and a final digit or X, and it must pass the modulus-11 check. ISBN-13 must contain 13 digits, use an ISBN prefix (978 or 979, excluding a non-ISBN 9790/ISMN prefix), and pass its modulus-10 check. Treat all ISBN data as text, not numeric quantities, so leading zeros are preserved. A valid check digit establishes mathematical validity, not that an ISBN belongs to the stated book.

## My decisions for ISBN values

1. **Keep a book when its ISBN is unusable.** If an ISBN cannot be safely validated or repaired, store NULL for that field, keep the book record when its book identity is usable, and log the original value and a warning. Preserve another ISBN in the same row if that other value is valid and not in conflict.

2. **Restore missing leading zeros.** For a digit-only ISBN-10 shorter than 10 digits, add zeros to the left until it has exactly 10 characters. Use it only if it passes the ISBN-10 check-digit test. Log the original value and result. Do not add zeros to arbitrary nonnumeric strings, or silently change a number that still fails validation.

3. **Clean simple formatting.** Remove hyphens and spaces in ISBN fields. For ISBN-10, change a final lowercase x to uppercase X. Validate afterward. If still invalid, store NULL and warn. Log all successful formatting repairs. Do not remove other unexpected letters or arbitrary extra digits.

4. **Handle scientific-notation ISBN-13 carefully.** Do not treat the expanded scientific-notation number as a complete ISBN-13 because it may have lost digits. Rebuild the ISBN-13 only from a validated ISBN-10 in the same incoming row, and only if the digits still represented by the scientific-notation value are consistent with that rebuilt ISBN-13. Compare the explicitly represented significant digits, not rounding artifacts or added zeros. Record the original value and rebuilt result. If there is no supporting ISBN-10 or the values conflict, store NULL for the damaged ISBN-13 and warn. Never invent an ISBN-13 from the scientific notation alone.

5. **Do not guess check digits.** If an ISBN fails its check-digit test, do not change the final character just to make it pass. Store NULL for the invalid value, keep the book and any other independently valid ISBN, and log the error. Only the specific, already approved conversions in Rules 4 and 6 may calculate check digits.

6. **Handle missing ISBNs.** If a valid ISBN-10 exists and the ISBN-13 field is empty, calculate its 978 ISBN-13 equivalent and record the derivation. If a valid ISBN-13 starts with 978 and ISBN-10 is empty, calculate its ISBN-10 equivalent and record the derivation. Do not calculate an ISBN-10 for an ISBN-13 beginning with 979: its absence is normal and does not require a missing-ISBN warning. If neither ISBN is usable, store NULL in both fields and log a warning; still keep the book when its identity is usable.

7. **Identify books by supplier book_id.** The supplier's book_id is the main key for detecting a book already in the database, including books with no usable ISBN. Compare title and author as a cross-check. Exact re-sends of a book are skipped rather than inserted again. A reused book_id with conflicting title or author must be flagged for review; do not automatically change the identity of an existing book. Do not automatically merge different book_id values, even when title or ISBN looks the same. Flag potential duplicates across different IDs.

8. **Accept only safe updates.** If the same book arrives again with an improved, validated ISBN and its identity agrees, fill previously missing ISBN information or replace information previously known to be invalid. Never erase a good stored value because a later file is blank or invalid. If the new ISBN conflicts with a different existing valid ISBN, preserve the existing value and flag the conflict instead of overwriting it. If there is no permitted improvement, classify the incoming row as skipped. Log every actual update.

9. **Produce a detailed report for every run.** Each execution must append a separate entry to ingestion-log.json, creating it if needed and preserving all earlier run entries. Each entry must show the timestamp, source filename, total data rows read, number of newly loaded, updated, skipped, and rejected rows, and a reason for every row that was not loaded. Record the CSV row number and book_id where available. Record the old and new ISBN values for repairs and updates, warnings for books imported with NULL ISBNs, conflicts, and what a person should review. A row loaded with a warning still counts as loaded; a previously existing row that was updated counts as updated, not newly loaded. Each source row must be counted in exactly one of the four outcome totals, even when it has more than one warning. The counts must add up to rows read. report.html must open without a server, let me select ingestion-log.json, and show the runs newest first, including problem rows and actions needed.

10. **Correct an ISBN in the wrong field.** If a mathematically valid ISBN-13 is found in the ISBN-10 field, or vice versa, place it in the correct field only when that destination is empty or otherwise nonconflicting. Derive the matching counterpart when allowed by Rule 6. Record the move. If the other field contains a different valid value, do not choose one automatically; treat it as a conflict under Rule 11.

11. **Do not accept conflicting ISBNs as verified.** If the incoming ISBN-10 and ISBN-13 are both individually valid but do not correspond to the same identifier, keep a new book by its book_id but store NULL in both ISBN fields and flag the row for review. If this is a repeat of an existing book, preserve its already validated values; do not overwrite them with a conflicting pair. If an incoming ISBN is already assigned to a different book_id, preserve that existing assignment and store NULL for the incoming conflicting ISBN; keep the incoming book record, and report the conflict. Never assume matching checksum digits prove matching book identities.

12. **Reject bad identifiers, not otherwise usable books.** Treat non-ISBN product codes, illegal characters, extra digits, damaged .0 decimal values, and unrecoverable ISBNs as invalid ISBNs. Do not guess, remove arbitrary digits, or treat other barcodes as ISBNs. Store NULL in the affected ISBN field, keep the otherwise identifiable book, and log the original value and exact problem. If the companion ISBN is valid and not in conflict, preserve it.

13. **Reject rows missing essential book information.** Every row must have a valid supplier book_id, a nonempty title, and a nonempty author after trimming surrounding whitespace. A book_id must be a positive whole number (not a decimal, negative number, blank, or other text). If any essential field is missing or invalid, reject the entire row, do not create or modify a database record, and log the CSV row number, original values, and exact reason. Keep otherwise identifiable books even when their ISBNs are unusable, as Rule 1 requires.

14. **Preserve all 23 supplier columns on new records.** When inserting a new book, keep the information from all 23 original CSV columns, subject to the ISBN validation rules and required identity checks. If the same `book_id` arrives later, change only validated ISBN fields when Rule 8 permits a safe improvement. Report differences in other fields for review, but do not automatically replace those fields. Never modify either source CSV file.

15. **Compare title and author without guessing.** For an incoming `book_id` that already exists, compare the title and author after removing leading and trailing whitespace. Do not automatically reword a title, infer that different names refer to the same book, or change the existing book's title or author. If the values still differ, flag the identity conflict for manual review and preserve the existing record.

## How I want the overall import to behave

- Process every CSV row on every run, using these same rules. Read ISBN values as strings and do not use an AI model or external lookup to make row-by-row decisions.
- Use books.db as the database and book_id as the unique book identity. A row must never create a duplicate record for an existing book_id.
- For existing books, compare validated/normalized ISBN values, rather than treating a previously repaired raw value as a new change. Repeating the first file should create no additional books and no unnecessary updates.
- Keep original CSV files unchanged. Keep original problematic identifiers in the run log even when the database stores NULL or a repaired value.
- Treat conflicting title/author changes on an existing book_id as review items, not automatic identity updates. Preserve existing records while the conflict is reviewed.
- Do not silently drop input rows. Every row must have an outcome in the report. If an input cannot be processed as a usable book, report it as rejected and explain why.
- The program should accept the input CSV filename from the terminal, so I can run it first with books.csv, again with books.csv, and then with books-batch2.csv.

## Clarifications I have confirmed

1. A validated ISBN-10 includes values safely repaired by Rules 2 or 3, provided they pass ISBN-10 check-digit validation.

2. Keep review flags, warnings, repairs, and reasons in ingestion-log.json rather than adding extra columns to books.db.

3. Use one books table containing the 23 supplier columns, with book_id as the primary key.

4. Check ISBN conflicts against original, repaired, and calculated ISBNs, including books already processed earlier in the same file.

5. I accept that books-batch2.csv may produce zero updated rows after books.csv has already repaired the available ISBNs. Do not force updates or change the source data. Keep Example 22 as a clearly labeled hypothetical example showing how a safe update would work on different input data. No additional test CSV is needed at this stage.

## Final answers to review questions

Numbered to match the nine review questions Claude asked. These are my final answers.

1. Scientific notation: Use the rounded comparison method. Reconstruct ISBN-13 only when the validated ISBN-10 produces an ISBN-13 consistent with the original scientific-notation value when rounded to the same precision. Log every repair. Do not use the more permissive within-one-unit method.

2. Recalculating rejected ISBN-13s: If Rule 4 rejects a damaged ISBN-13 because it is inconsistent with the valid ISBN-10, leave ISBN-13 as NULL and log a warning. Do not use Rule 6 to bypass that rejection. Rule 6 may still derive ISBN-13 when the original field was genuinely empty.

3. Title field: Use `title`, not `original_title`. A missing `original_title` is acceptable.

4. Title and author comparison: Remove leading and trailing spaces for comparison, then use exact, case-sensitive matching.

5. Conflicting identity: If an existing `book_id` has a different title or author, reject the incoming row, preserve the existing record, and report the conflict.

6. Other supplier columns: Preserve original values as text, except for `book_id`, which must be a valid integer primary key. Store blank cells as NULL. Preserve original title and author text when inserting. Use trimming only for comparisons. Report changes to non-ISBN fields without automatically updating them.

7. Potential duplicates: Different `book_id` values with matching titles and authors should remain separate records, but receive a potential-duplicate warning. Apply the established ISBN conflict rules.

8. Row numbers: Report actual CSV line numbers, beginning with line 2 for the first data record.

9. HTML report: Create a self-contained `report.html` with a file-selection button to open `ingestion-log.json`. Show all runs newest first without requiring a server.

## Final decisions on the remaining questions

1. **Scientific-notation rounding:** Use round half to even (banker's rounding). Compare the reconstructed ISBN-13 against the scientific-notation value using the number of significant digits shown in the original CSV field. Use deterministic decimal arithmetic, not approximate floating-point comparisons or a within-one-unit tolerance. Only accept the reconstruction when it matches under this rule.

2. **Spaces-only cells:** Treat cells containing only whitespace as empty and store NULL. Preserve the original CSV files unchanged.

3. **Title and author spacing:** Ignore leading and trailing whitespace when comparing title and author values. If the trimmed values match exactly, do not report a field change for whitespace alone. Preserve the originally stored information.

4. **Rejected ISBN confirmation:** I confirm your interpretation. If an ISBN value is rejected under Rule 4 or Rule 12, leave that field NULL and do not reconstruct it using another ISBN field. Derivation is permitted only when the original ISBN field was genuinely empty.

## Test examples — expected behavior

These are the examples I will use to check the finished program. NULL means an SQL missing value, not the text "NULL". These examples describe ISBN handling; whether an entire source row is new, updated, or skipped also depends on the existing database state.

| # | Input ISBN value / situation | Expected behavior |
|---|---|---|
| 1 | ISBN-10 `0439139600` | Accept as valid, unchanged. |
| 2 | ISBN-13 `9780439139601` | Accept as valid, unchanged. |
| 3 | ISBN-10 `439023483` | Pad left to `0439023483`, validate, accept, and log repair. |
| 4 | ISBN-10 `61120081` | Pad left to `0061120081`, validate, accept, and log repair. |
| 5 | ISBN-10 `0-450-19827-8` | Remove hyphens, validate `0450198278`, and log cleanup. |
| 6 | ISBN-10 `162 001 4750` | Remove spaces, validate `1620014750`, and log cleanup. |
| 7 | ISBN-10 `031200009x` | Change final x to X, accept `031200009X` after validation, and log cleanup. |
| 8 | ISBN-10 `0439023483` and ISBN-13 `9.78043902348e+12` | Derive `9780439023481` from ISBN-10, confirm represented prefix matches, and log reconstruction. |
| 9 | ISBN-13 `9.78043902348e+12` with no valid ISBN-10 | Store NULL for ISBN-13; do not guess final digits; warn. |
| 10 | ISBN-10 `1854433017` (Small Hours) | Failed check digit: store NULL, keep book, log warning, do not simply change final digit. |
| 11 | ISBN-10 `04390A3483` (The Glass Aviary) | Illegal A: store NULL, keep book, log warning. |
| 12 | ISBN-10 `15287463097` (Borrowed Light) | 11 digits: do not drop a digit; store NULL and warn. |
| 13 | ISBN-13-like code `2940016231920` | Non-ISBN prefix: store NULL and warn. |
| 14 | ISBN-13-like value `195170342.0` | Invalid decimal form: store NULL and warn. |
| 15 | ISBN-13 `9798351220413`, ISBN-10 blank | Accept valid 979 ISBN-13; leave ISBN-10 NULL without a missing-ISBN warning. |
| 16 | ISBN-13 `9780062398123`, ISBN-10 blank | Accept ISBN-13; derive ISBN-10 `0062398121`; log derivation. |
| 17 | Both ISBN fields blank (Untitled Manuscript) | Load identifiable book with both ISBNs NULL; warn about missing source ISBNs. |
| 18 | ISBN-13 `9781982147600` in the isbn field; isbn13 blank | Move valid ISBN-13 to its correct field; derive ISBN-10 `1982147601`; log both actions. |
| 19 | ISBN-10 `1555550126` with ISBN-13 `9781555550134` (Tides of Marrow Bay) | Each checks out individually but the pair disagrees: for new book, store both as NULL, keep book, flag review. |
| 20 | ISBN-10 `1416524797` used for Paper Moons when already assigned to Angels & Demons | Preserve prior assignment, store NULL for incoming conflicting ISBN, keep incoming book, flag review. |
| 21 | Exact repeated row for book_id=10001, The Lantern Keeper's Daughter | Insert at most once; classify the repeat as skipped, report duplicate. |
| 22 | **Hypothetical (see Clarification 5):** Existing book_id=24 returns with a validated ISBN replacing a prior missing/invalid one | Update only a safe missing/invalid value; preserve good values; log update. |
| 23 | Existing book_id=25 returns with both ISBNs blank | Do not erase existing valid ISBNs; classify as skipped if nothing improves; warn as appropriate. |
| 24 | Missing, nonnumeric, or nonpositive book_id, or blank title or author | Reject entire row; do not write to database; log original values, CSV row number, and specific missing/invalid field. |
| 25 | Existing `book_id` with unchanged title/author and a different value in a non-ISBN supplier column | Leave the stored non-ISBN field unchanged; report the difference for review. Update only a safe ISBN improvement if one exists. |
| 26 | Existing `book_id` with title `The Hunger Games ` (trailing space) and otherwise unchanged author | Trim surrounding spaces for comparison and recognize the existing book rather than creating a duplicate. |
