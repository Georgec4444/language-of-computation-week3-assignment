# Language of Computation – Week 3 Assignment

This is my Module 3 assignment for MSAI-5523, Language of Computation.

## Files

| File | Purpose |
|---|---|
| `books.csv`, `books-batch2.csv` | Supplier book files. They are only read, never changed. |
| `data-validation.md` | Diagnosis of the ISBN values in both CSV files. |
| `rules.md` | My approved business rules for importing books. |
| `ingest.py` | Imports one CSV file into `books.db`, following `rules.md`. |
| `books.db` | SQLite database (created by the first run). |
| `ingestion-log.json` | One entry per run, appended every time (created by the first run). |
| `report.html` | Opens in a browser and shows `ingestion-log.json`, newest run first. |

## How to run it in Windows PowerShell

### 1. Open PowerShell in the assignment folder

1. Press the **Windows** key, type `PowerShell`, and press **Enter**.
2. Go to the assignment folder:

   ```powershell
   cd "C:\Users\GC4Ne\workspace\language-of-computation\week3-assignment"
   ```

3. Check that Python works:

   ```powershell
   python --version
   ```

   It should print `Python 3.13.17`. If it opens the Microsoft Store or says Python was not found, use `py` instead of `python` in the commands below.

### 2. Run `ingest.py` on each CSV file

Run these one at a time, in this order:

```powershell
python ingest.py books.csv
```

```powershell
python ingest.py books.csv
```

```powershell
python ingest.py books-batch2.csv
```

Each run prints one summary line: rows read, and how many were loaded, updated, skipped, and rejected. It also adds a new entry to `ingestion-log.json` without changing earlier entries. `books.db` and `ingestion-log.json` are always written in the same folder as `ingest.py`.

### 3. Open the report

```powershell
Start-Process .\report.html
```

The report opens in your default browser. Click **Choose File**, select `ingestion-log.json` in this folder, and every run is shown newest first. Each run shows its counts and every row that was not loaded, or that had warnings, corrections, or conflicts. Use the search box, the outcome list, and **Needs review only** to filter the rows. The page works without a server or internet connection.

### Optional: start over with an empty database

Only if you want to repeat the runs from the beginning, delete the database and the log first. This permanently removes all earlier import results:

```powershell
Remove-Item .\books.db, .\ingestion-log.json
```
