# Lab 5 – Reflection

**Name:** Prathith Shetty  **SRN:** PES2UG24AM121  **File:** `vehicle_service.py`

### 1. Which issues were the easiest to fix, and which were the hardest? Why?

**Easiest:** the Flake8 style issues (E231, E225, E302, E501) and Pylint's missing encoding (W1514). The message names the exact line and the exact change: add a space, add a blank line, wrap a long line, pass `encoding="utf-8"`. None of them can change what the program does.

**Hardest:** the MD5 warning (Bandit B324). The obvious fix, switching to SHA-256, would change every job card id the program prints (`353ee89f` would become a different value) and break ids already given to customers. I had to work out what the hash is actually used for. It only builds a short id, not a password or signature, so the correct fix was `usedforsecurity=False`. The snake_case renames (C0103) were also more work than they looked: every rename meant updating the call sites in `main()` and re-running the program to confirm the output did not change. The mutable default argument (W0102) was easy to change but harder to understand: Python evaluates `history=[]` only once, when the function is defined, so the same list is shared by every call.

### 2. Did the static analysis tools report any false positives? If so, describe one example.

Yes, partly:

- **Bandit B324 (MD5)** is a false positive in this context. Bandit cannot tell that the hash only builds a non-secret job card id, so it treats it as a security use. Declaring `usedforsecurity=False` documents that intent and clears the warning without changing behaviour.
- **Bandit B404 (import subprocess)** is informational rather than a real vulnerability. Importing `subprocess` is not unsafe on its own; the actual problem was `shell=True` with string concatenation (B602). After fixing B602 the import was no longer needed, so I removed it.
- **Pylint C0103 (naming)** is a convention, not a bug. camelCase names work fine, but they don't match PEP 8, so I still fixed them for consistency.

The B105 hardcoded password was **not** a false positive. Even though `ADMIN_PASSWORD` is unused, the password is still in the source and the Git history.

### 3. How would you integrate static analysis tools into your actual software development workflow?

- **While coding:** run Pylint and Flake8 inside the editor (VS Code extensions), so problems are underlined as I type instead of being found at the end.
- **Before each commit:** a `pre-commit` hook that runs `flake8`, `bandit -ll` (medium and high only) and `pylint --fail-under=9.5`, so badly styled or insecure code cannot be committed.
- **In CI:** a GitHub Actions workflow that runs all three tools on every push and pull request, saves the reports as artifacts, and blocks merging on any High-severity Bandit issue or a Pylint score below the threshold.
- **Shared configuration:** keep the rules in `pyproject.toml` / `setup.cfg` and pin the tool versions, so every developer and the CI server get the same results. Pinning matters: on my machine Pylint crashed until I reinstalled `isort` as pure Python, and a fixed, tested environment in CI avoids surprises like that.

### 4. What tangible improvements did you observe in the code quality, readability, or potential robustness after applying the fixes?

- **Measured:** Pylint went from **4.66/10 to 10.00/10**, Bandit from **4 issues (2 High) to 0**, and Flake8 from **4 issues to 0**. The demo prints exactly the same output as before.
- **Security:** `backup_records` can no longer be used for shell command injection, and it now also works on Windows, because `shutil.copy` replaces the Unix-only `cp`. The admin password is no longer stored in the code.
- **Robustness:** `add_service` no longer leaks bookings between calls through a shared default list (I checked that two separate calls now return separate histories). `cancel_service` only handles the expected `KeyError`, so real bugs are no longer hidden behind "No records". `services.json` is written and read with the same UTF-8 encoding on every OS.
- **Readability and reuse:** consistent snake_case names and docstrings make each function's purpose clear. The `if __name__ == "__main__":` guard means the module can be imported, for example by tests, without running the demo or writing files.
