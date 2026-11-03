# Data Validation: ISBN Diagnosis

MSAI-5523 Module 3, Part 2. This diagnoses the `isbn` and `isbn13` columns of `books.csv` and `books-batch2.csv`. No CSV values were changed and nothing was fixed; this is a description of what is in the files today.

- **books.csv**: 10,000 data rows, downloaded from <https://raw.githubusercontent.com/zygmuntz/goodbooks-10k/master/books.csv>
- **books-batch2.csv**: 31 data rows (already in the folder)
- **Reference**: International ISBN Agency, *ISBN Users' Manual*, 7th edition (<https://www.isbn-international.org/content/isbn-users-manual/29>)

"Line" in the examples means the line number in the CSV file. Line 1 is the header, so the first book is line 2.

---

## 1. What makes an ISBN valid

The ISBN Users' Manual defines two forms.

**ISBN-10** (used before 2007):

- Exactly 10 characters: 9 digits followed by a check digit.
- The check digit may be `X`, which stands for the value 10. `X` is allowed only in the last position and never anywhere else.
- **Check-digit rule (modulus 11):** multiply the 10 characters by 10, 9, 8, … 1 and add the results. The total must divide evenly by 11.
- Worked example: `0439139600` gives 0×10 + 4×9 + 3×8 + 9×7 + 1×6 + 3×5 + 9×4 + 6×3 + 0×2 + 0×1 = 198. 198 = 11 × 18, so it is valid.

**ISBN-13** (the current form):

- Exactly 13 digits.
- It starts with the prefix `978` or `979`.
- **Check-digit rule (modulus 10):** multiply the digits by 1, 3, 1, 3, … in turn and add the results. The total must divide evenly by 10.

**How the two forms relate:**

- To turn an ISBN-10 into an ISBN-13, put `978` in front of its first 9 digits and **recalculate** the check digit. The old check digit cannot be reused.
- An ISBN-13 that starts with `979` has **no** ISBN-10 equivalent.

**Hyphens and spaces** (for example `0-450-19827-8`) are only there to make the number easier to read. They are not part of the ISBN itself.

**Leading zeros are part of the ISBN.** `0439023483` and `439023483` are not the same value.

**Important limit:** a check digit tells you whether a number is *well-formed*. It does not tell you whether the number belongs to *this* book. A typo that happens to pass the check, or another book's ISBN pasted into the wrong row, will still pass.

---

## 2. Summary

### books.csv (10,000 rows)

**`isbn` column:**

| Result | Rows |
|---|---:|
| Valid as stored | 2,690 |
| Becomes valid once the missing leading zeros are added back (problem A) | 6,587 |
| Empty (problem B) | 700 |
| Too short, and still invalid even with zeros added (problem C) | 14 |
| 10 digits but wrong check digit (problem D) | 9 |
| **Total** | **10,000** |

**`isbn13` column:**

| Result | Rows |
|---|---:|
| Valid as stored | **0** |
| Stored in scientific notation, e.g. `9.78043902348e+12` (problem E) | 9,410 |
| Empty (problem B) | 585 |
| Stored as a decimal with a `.0` ending, e.g. `195170342.0` (problem F) | 5 |
| **Total** | **10,000** |

**In plain English:**

- Only about 27% of the ISBN-10 values in `books.csv` are usable exactly as stored.
- **None** of the ISBN-13 values are, because every one has been damaged by number formatting.

### books-batch2.csv (31 rows)

**`isbn` column:**

| Result | Rows |
|---|---:|
| Valid as stored | 12 |
| Becomes valid once leading zeros are added back (problem A) | 7 |
| Valid once formatting is cleaned (hyphens, spaces, lowercase `x`) (problem G) | 3 |
| Empty (problem B) | 5 |
| Contains a letter that is not allowed (problem H) | 1 |
| Wrong check digit (problem D) | 1 |
| Wrong length: an ISBN-13 placed in the ISBN-10 column (problem I) | 1 |
| Wrong length: 11 digits (problem I) | 1 |
| **Total** | **31** |

**`isbn13` column:**

| Result | Rows |
|---|---:|
| Valid as stored | 9 |
| Scientific notation (problem E) | 14 |
| Empty (problem B) | 8 |
| **Total** | **31** |

**Problems in batch2 that only show up when comparing rows or fields** (section 4):

| Problem | Rows affected |
|---|---:|
| `isbn` and `isbn13` both valid but describe different books | 1 |
| One book's ISBN copied onto a different book | 1 |
| Exact duplicate row | 1 extra copy |
| `book_id` already exists in `books.csv` | 16 |

---

## 3. Problems found in single values, one at a time

### A. Leading zeros dropped from the ISBN-10

**Rows:**

| Digits remaining | books.csv | books-batch2.csv |
|---|---:|---:|
| 9 (one zero lost) | 5,563 | 6 |
| 8 (two zeros lost) | 913 | 1 |
| 7 (three zeros lost) | 111 | 0 |
| **Total** | **6,587** | **7** |

**What is wrong:** the value is shorter than 10 digits because the zeros at the start are gone.

**How I know it's this:** add zeros back to the front until the value is 10 digits long, and every one of these passes the modulus-11 check. Random short numbers would almost never do that.

**Examples:**

- `books.csv` line 2, *The Hunger Games*: `439023483` should be `0439023483`.
- `books.csv` line 5, *To Kill a Mockingbird*: `61120081` should be `0061120081`.
- `books.csv` line 70, *Insurgent*: `7442912` should be `0007442912`.
- `books-batch2.csv` line 6, *The Long Walk Home*: `441773028` should be `0441773028`.

**Probable cause:** the column was treated as a **number** instead of **text**, by a spreadsheet such as Excel or by a CSV library that guesses data types. Numbers don't keep leading zeros, so `0439023483` became `439023483`. Many English-language publishers' ISBNs start with `0` (registration group 0), which is why more than half the rows are hit.

### B. Empty values

**Rows:**

| | books.csv | books-batch2.csv |
|---|---:|---:|
| `isbn` empty | 700 | 5 |
| `isbn13` empty | 585 | 8 |
| Both empty | 565 | 3 |

**What is wrong:** there is no value. An empty value is not always an *error*, though. Some cases are legitimate:

- **No ISBN-10 can exist for a `979` ISBN.** `books-batch2.csv` line 10, *Notes from the Night Ferry*, has `isbn13 = 9798351220413` and an empty `isbn`. That is correct. Nothing is missing.
- **An unpublished work has no ISBN yet.** `books-batch2.csv` line 22, *Untitled Manuscript* by "Unknown", has both fields empty. That is probably a placeholder record, not a published book.
- **The missing form can be calculated from the other one.** `books-batch2.csv` line 25, *The Cartographer's Error*, has an empty `isbn` but a valid `isbn13 = 9780062398123`. Its ISBN-10 is `0062398121`.
- **Missing source data.** In `books.csv` line 107, *Bossypants*, both are empty even though it is a well-known published book. The original data simply didn't record an ISBN, often because Goodreads lists the e-book or audiobook edition, which may not have one.

**Probable cause:** the data was missing at the source, the book is an e-book or audiobook edition, the ISBN-13 is a `979` one with no ISBN-10, or the book is unpublished.

### C. Too short, and still invalid after adding zeros back

**Rows:** 14 in `books.csv` (10 nine-digit, 3 eight-digit, 1 seven-digit). 0 in batch2.

**What is wrong:** the leading zeros were lost (problem A), *and* the remaining digits are wrong. For most of these rows the ISBN-13 column shows that **one digit was mistyped**.

**Examples:**

- `books.csv` line 917, *Reading Lolita in Tehran*: `isbn = 812971060`, padded to `0812971060`. That fails the check.
  - Its `isbn13` reads `9.78081297106e+12`, which points to ISBN-10 `081297106X`.
  - So the final `X` was lost, and the value was then treated as a number.
- `books.csv` line 1096, *Life As We Knew It*: `isbn = 152061548`. It fails the check, and `isbn13` is empty, so there is nothing to compare it with.

**Probable cause:**

- Converting to a number destroys a final `X` (a number can't contain a letter), so ISBN-10s ending in `X` lose that character.
- Other rows have a simple typing or copying error on top of the lost zeros.

### D. 10 characters, but the check digit is wrong

**Rows:** 9 in `books.csv`, 1 in batch2.

**What is wrong:** the format looks right, but the modulus-11 total doesn't divide evenly by 11.

**What I found in books.csv:** in **8 of the 9 rows**, the bad ISBN-10 is exactly the **last 10 digits of the book's ISBN-13**.

- Example: `books.csv` line 2779, *Two-Way Street*: `isbn = 1416913184`. The ISBN-13 is `9781416913184`.
- Someone (or some program) made the ISBN-10 by cutting `978` off the front. That keeps the ISBN-13's check digit (`4`) instead of recalculating it. The correct ISBN-10 is `1416913181`.

**Examples:**

- `books.csv` line 1444, *The Secret of the Nagas*: `9380658797`. It should end in `6`. This row's `isbn13` is also damaged (see problem F).
- `books-batch2.csv` line 11, *Small Hours*: `1854433017`. The correct check digit for `185443301` is `6`.
  - Swapping two neighbouring digits (`1845433017` or `1854343017`) would also make it pass. So this could be a **transposition typo** rather than a bad check digit.
  - The file alone can't tell which digit is wrong.

**Probable cause:**

- In `books.csv`: a faulty ISBN-13 to ISBN-10 conversion that skipped recalculating the check digit.
- In batch2: a typing mistake.

### E. ISBN-13 stored in scientific notation

**Rows:** 9,410 in `books.csv` (every non-empty `isbn13` except 5). 14 in batch2.

**What is wrong:** `9780439023481` was saved as `9.78043902348e+12`.

- Scientific notation here keeps only about **12 significant digits**. A 13-digit ISBN therefore loses its last digit, which is the **check digit**.
- No scientific-notation value can be checked on its own. `9.78043902348e+12` reads back as `9780439023480`, which fails the check.
- Shorter forms such as `9.78159448e+12` (*The Kite Runner*) look like they lost more, but only zeros were trimmed. They are still rounded to the same 12 digits.

**Can the true value be recovered?** Sometimes, by working it out from the row's ISBN-10. In `books.csv`:

| Situation | Rows |
|---|---:|
| Rounded value matches the ISBN-13 worked out from a valid (or zero-padded) ISBN-10 | 9,239 |
| Rounded value **does not** match that ISBN-10 (they describe different editions or different books) | 15 |
| No valid ISBN-10 in the row, so there is nothing to recover from | 156 |

**Mismatch examples:**

- `books.csv` line 623, *Nights in Rodanthe*: `isbn` is `446612707` (`0446612707`), but `isbn13` is around `9780553813950`. That is a different publisher prefix, so probably a different edition.

**Values that aren't ISBNs at all:** 28 scientific-notation values in `books.csv` don't start with `978` or `979`.

- 22 start with `2940…`, for example line 1166, *Wait for You*: `2.94001623192e+12`. That pattern matches the e-book product codes that stores such as Barnes & Noble assign. It is not an ISBN.
- Others are 12-digit numbers such as `6.45573003012e+11` (line 2895, *Sailor Moon, #1*). Those look like UPC barcodes, not ISBNs.

**Prefix 979-0:** one value starts with `9790` (line 1107, *The Great Divorce*). `979-0` is reserved for the ISMN, the music-publication number, so this is very likely an error too.

**Probable cause:**

- A spreadsheet (most commonly Excel) or a program that treated a 13-digit code as a large number and saved it with default number formatting.
- Goodreads also stored non-ISBN product codes in the ISBN-13 field.

### F. Damaged decimal values ending in `.0`

**Rows:** 5 in `books.csv`. 0 in batch2.

**What is wrong:** the value was turned into a decimal number (`.0` added), *and* it has only 9 to 11 digits, so it can't be a 13-digit ISBN.

**Examples:**

- Line 8175, *Washington's Crossing*: `195170342.0`. That is the ISBN-10 (missing its leading zero) placed in the ISBN-13 column.
- Line 1444, *The Secret of the Nagas*: `97893806587.0`. These are the first 11 digits of an ISBN-13.
- Line 959, *The Complete Anne of Green Gables Boxed Set*: `76783609419.0`. That is an 11-digit number that looks like a UPC barcode code, not an ISBN.

**Probable cause:** the same number conversion as problems A and E, applied to values that were already wrong or were the wrong kind of code.

### G. Valid once formatting is cleaned up

**Rows:** 3 in batch2. 0 in `books.csv`.

**What is wrong:** the digits are correct, but the value contains characters that aren't part of the ISBN, or uses a lowercase letter.

| Line | Book | Stored | Clean value | Problem |
|---|---|---|---|---|
| 30 | *Salt and Ember* | `0-450-19827-8` | `0450198278` | hyphens |
| 31 | *A Field Guide to Quiet Places* | `" 162 001 4750 "` | `1620014750` | spaces inside the value and at both ends |
| 27 | *Winter Orchard* | `031200009x` | `031200009X` | lowercase `x` |

**Probable cause:** the values were typed or pasted by hand from a printed book or a website, where ISBNs are shown with hyphens or spaces for readability. The manual says these separators are for display only.

### H. Illegal character

**Rows:** 1 in batch2.

**Example:** line 3, *The Glass Aviary*: `04390A3483`.

**What is wrong:** an ISBN-10 may contain only digits, plus `X` in the last position. An `A` in the middle is never allowed.

**Interesting detail:** if the `A` were a `2`, the value would be `0439023483`. That is *The Hunger Games'* ISBN, not this book's.

**Probable cause:** a typo, or a corrupted copy of another book's ISBN.

### I. Wrong length, or the wrong kind of ISBN in the column

**Rows:** 2 in batch2.

**What is wrong:**

- Line 18, *An Atlas of Small Kindnesses*: `isbn = 9781982147600`.
  - That is a **valid ISBN-13 in the ISBN-10 column**, and the `isbn13` column is empty.
  - Its ISBN-10 form would be `1982147601`.
  - **Probable cause:** the value was entered into the wrong column.
- Line 20, *Borrowed Light*: `isbn = 15287463097`, which is 11 digits.
  - It is a valid ISBN-10 (`1528746309`) with an extra `7` on the end.
  - That ISBN-10 also belongs to a different book in the same file (*The Lantern Keeper's Daughter*).
  - **Probable cause:** an extra keystroke while copying, plus the wrong book's ISBN.

---

## 4. Problems that only appear when comparing values

Each value in this section passes the format and check-digit rules **on its own**. It is wrong only when compared with other fields or rows. Single-value checks will not catch these.

### J. `isbn` and `isbn13` disagree

**Row:** `books-batch2.csv` line 16, *Tides of Marrow Bay*: `isbn = 1555550126`, `isbn13 = 9781555550134`.

- Both pass their own check.
- But `1555550126` converts to `9781555550127`, not `…134`. The two fields describe different books.
- **Probable cause:** a typo in one field, with the check digit then recalculated, or two editions mixed up.

In `books.csv`, the 15 mismatches between ISBN-10 and the rounded ISBN-13 values are listed under problem E.

### K. Same ISBN used for two different books

**Row:** `books-batch2.csv` line 24, *Paper Moons* (Georgia Tran), `book_id` 10013, has `isbn = 1416524797`.

- That is the ISBN of *Angels & Demons* by Dan Brown: `book_id` 9 in both files.
- An ISBN identifies exactly one edition of one book, so this is wrong even though the check digit passes.
- **Probable cause:** a copy-and-paste error from the row above or below.

No ISBN-10 is shared by two different books *within* `books.csv`.

### L. Duplicate rows and overlap with books.csv

- **Exact duplicate:** `books-batch2.csv` lines 14 and 15 are the same row: `book_id` 10001, *The Lantern Keeper's Daughter*.
- **Overlap:** 16 of the 31 `book_id`s in batch2 already exist in `books.csv`.
  - For 14 of them, the ISBN values are exactly the same, including the same damage (for example *The Hunger Games*, `439023483`, `9.78043902348e+12`). They are re-sends of existing books.
  - `book_id` 24, *Harry Potter and the Goblet of Fire*:
    - `books.csv` has `439139600` / `9.7804391396e+12`.
    - batch2 has the full, correct `0439139600` / `9780439139601`.
    - Same book, but the batch2 version is the *better* value.
  - `book_id` 25, *Harry Potter and the Deathly Hallows*:
    - `books.csv` has `545010225` / `9.78054501022e+12`.
    - batch2 has **empty** values for both.
    - Batch2 is *worse* here and would wipe out data if it replaced the original.
- **Probable cause:** batch2 is a mix of new books (`book_id`s 10001–10014) and records re-exported from the main dataset, sometimes cleaned and sometimes not.

---

## 5. Root causes in plain English

1. **Treating ISBNs as numbers instead of text.** This one mistake explains most of the damage:
   - leading zeros lost (A, C)
   - final `X` lost (C)
   - scientific notation and the lost check digit (E)
   - the `.0` endings (F)

   It affects nearly every row of `books.csv`. An ISBN is an *identifier* made of digits, like a phone number. It is not a quantity, and you never do arithmetic on it.
2. **Manual data entry.** Hyphens, spaces, lowercase `x`, a stray letter, an extra digit, the wrong column, and transposed digits (D, G, H, I).
3. **Faulty ISBN-13 to ISBN-10 conversion.** Cutting off `978` without recalculating the check digit (D in `books.csv`).
4. **Copy-and-paste between rows.** One book's ISBN on another book, and duplicate rows (H, I, K, L).
5. **Non-ISBN codes stored as ISBNs.** E-book store codes (`2940…`) and UPC barcodes in the ISBN-13 column (E, F).
6. **Missing data that may be legitimate.** E-books or audiobooks without an ISBN, `979` ISBNs with no ISBN-10, and unpublished works (B).

---

## 6. Questions to answer when you write your Part 3 rules

These are decisions for you to make. I haven't applied any of them.

- **Read ISBN columns as text.** Should ingestion always read the ISBN columns as text so no further damage happens?
- **Restoring leading zeros:** is it acceptable to add zeros back to a short ISBN-10, *only* if the padded result passes the check digit? (That would rescue 6,587 + 7 rows.)
- **Cleaning formatting:** should you remove hyphens and spaces, and convert lowercase `x` to `X`, before checking?
- **Scientific-notation ISBN-13s:** these can't be trusted on their own. Should you rebuild the ISBN-13 from a valid ISBN-10 when the rounded value matches, and otherwise set it to empty or flag it for review?
- **Non-ISBN codes:** should values that don't start with `978`/`979` (and `9790`) be rejected?
- **Wrong check digits:** should you reject the value, or flag it with a suggested correction? Never correct it silently, because the wrong digit could be anywhere.
- **Wrong column:** should an ISBN-13 found in the ISBN-10 column be moved to the ISBN-13 column?
- **Calculating the missing form:** should a missing ISBN-10 or ISBN-13 be calculated from the other one when possible? (Never for `979`.)
- **Disagreeing pairs:** what happens when `isbn` and `isbn13` are both valid but don't match?
- **Shared ISBNs:** what happens when two different books share an ISBN?
- **Duplicates and re-sent rows:** how should exact duplicates be handled? And how should re-sent `book_id`s be handled, so that a worse value (such as an empty one) never overwrites a good one?
- **Empty values:** should empty be allowed, and should "unknown" be stored as a real empty value (NULL) rather than as an empty string?
