# Lab 5 – Static Code Analysis: Issues Found and Fixed

**Name:** Prathith Shetty  **SRN:** PES2UG24AM121  **File:** `vehicle_service.py`

Tools: Pylint 4.1.4, Bandit 1.9.4, Flake8 7.5.0 (Python 3.12).
Line numbers refer to the **original** `vehicle_service.py`.

## Summary

| Tool | Before | After | Reports |
|------|--------|-------|---------|
| Pylint | 4.66/10 (31 messages) | **10.00/10** (0 messages) | [before](reports/before/pylint_report.txt) · [after](reports/after/pylint_report.txt) |
| Bandit | 4 issues (2 High, 2 Low) | **No issues identified** | [before](reports/before/bandit_report.txt) · [after](reports/after/bandit_report.txt) |
| Flake8 | 4 issues | **0 issues** (empty report) | [before](reports/before/flake8_report.txt) · [after](reports/after/flake8_report.txt) |

The program prints exactly the same output before and after the fixes:

```
No records for KA09ZZ0001
Total for KA01AB1234: 2300
Total for KA05XY9876: 2500
Job card for KA01AB1234: 353ee89f
Vehicles on file: 3 | busiest vehicle: KA01AB1234
```

## Issue table (high and medium severity first)

| Issue | Type | Line(s) | Description | Fix Approach |
|-------|------|---------|-------------|--------------|
| Shell command injection – Bandit **B602** (High) | Security | 61 | `subprocess.call("cp services.json " + folder, shell=True)` passes a string built from `folder` to the shell, so a folder name such as `x; rm -rf ~` would run extra commands. `cp` also does not exist on Windows. | Replaced it with `shutil.copy("services.json", folder)`, which copies the file without going through a shell. |
| Weak hash – Bandit **B324** (High) | Security | 39 | `hashlib.md5(...)` is flagged because MD5 is broken for security use. Here it only builds a short job card id from the vehicle number. | Kept MD5 but declared `usedforsecurity=False` (Bandit's own recommendation), so existing job card ids stay the same (`353ee89f`). If the id were ever used as a secret, switch to `hashlib.sha256`. |
| Mutable default argument – Pylint **W0102** | Bug | 12 | `history=[]` is created once, when the function is defined, so every call to `addService` without a history appends to the same shared list, and the list keeps growing across unrelated bookings. | Changed the default to `None` and create a new list inside the function (`if history is None: history = []`). |
| Overly broad exception – Pylint **W0718** | Bug | 23 | `except Exception:` hides every error (for example a `TypeError` from a bad argument) and reports it as "No records". | Catch only `KeyError`, the one error expected when a vehicle has no records. |
| Open without encoding – Pylint **W1514** | Bug | 50 | `open(fileName, "w")` uses the platform default encoding (cp1252 on Windows), while `loadRecords` reads with UTF-8, so the file might not round-trip. | Added `encoding="utf-8"` when writing, matching the read. |
| Hardcoded password – Bandit **B105** (Low) | Security | 8 | `ADMIN_PASSWORD = "garage-demo"` stores a credential in source code and in Git history. | Read it from the `GARAGE_ADMIN_PASSWORD` environment variable instead (`os.environ.get(...)`). |
| Import of subprocess – Bandit **B404** (Low) | Security | 3 | Bandit warns that the `subprocess` module can run shell commands. | Removed the import; it is no longer needed after the B602 fix. |
| Naming convention – Pylint **C0103** (18 messages) | Convention | 12, 20, 31, 32, 34, 38, 42, 49, 54, 59 | camelCase names (`addService`, `vehicleNo`, `totalCost`, `fileName`, …) do not follow PEP 8 snake_case. | Renamed everything to snake_case (`add_service`, `vehicle_no`, `total_cost`, `file_name`, …) and updated every call in `main()`. |
| Missing docstrings – Pylint **C0114 / C0116** (10 messages) | Convention | 1, 12, 20, 31, 38, 42, 49, 54, 59, 64 | No module docstring and no function docstrings. | Added a module docstring and a docstring to every function. |
| Missing whitespace after ',' – Flake8 **E231** | Style | 7 | `"General": 2500,"Brake Check": 1200` | Added a space after the comma. |
| Missing whitespace around operator – Flake8 **E225** | Style | 43 | `best=None` | Changed it to `best = None`. |
| Expected 2 blank lines – Flake8 **E302** | Style | 49 | Only one blank line before `def saveRecords`. | Added the second blank line. |
| Line too long – Flake8 **E501** | Style | 77 | The `print(...)` in `main()` was 83 characters (limit 79). | Wrapped the call over two lines. |
| Demo runs on import (not flagged by the tools) | Design | 80 | `main()` was called at module level, so importing the module ran the demo and wrote `services.json`. | Wrapped it in `if __name__ == "__main__":`. |

## Commits

| Commit | What it fixes |
|--------|---------------|
| `Add baseline Pylint, Bandit and Flake8 reports (before fixes)` | Saves the three "before" reports |
| `Fix Bandit security issues` | B602, B324, B105, B404 |
| `Fix Flake8 style issues` | E231, E225, E302, E501 |
| `Fix Pylint issues (score 4.66 -> 10.00)` | W0102, W0718, W1514, C0103, C0114, C0116 |

## Note on running Pylint on Windows

On my laptop the first Pylint run crashed (`astroid-error`, rated 0.00/10): Windows Application Control blocked a compiled DLL inside Pylint's `isort` dependency. Reinstalling isort as pure Python fixed it:

```
python -m pip install --force-reinstall --no-deps --no-binary isort isort
```
