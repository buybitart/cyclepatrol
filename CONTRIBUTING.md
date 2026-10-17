# Contributing to CyclePatrol

Thanks for your interest in CyclePatrol. It is one Bash script, `cp.sh`, plus a vendor data set and
this documentation. Contributions that improve correctness, safety, and clarity are welcome.

CyclePatrol runs active wireless attacks and is for authorized testing only. Please keep that in
mind in every change: do not add a feature that makes unauthorized use easier or that weakens the
scope gate or the authorization defaults.

## Report a bug

Open a GitHub issue. A useful report includes:

- what you ran (the command line and any env vars),
- the OS, the adapter, and the `bash --version`,
- what you expected and what happened,
- the relevant lines from `cp_audit.log` or the session log (remove any real keys or BSSIDs you are
  not allowed to share).

## Suggest a change

Open an issue first for anything beyond a small fix, so the design can be agreed before you write
code. For a typo or a doc fix, a pull request on its own is fine.

## Prepare a pull request

1. Keep the change focused. One topic per pull request.
2. The tool is pure Bash 5 by design. Do not add a Python, Node, or other runtime dependency.
3. New runtime dependencies must be standard Linux wireless tools installable with `apt-get`, and
   each module must still skip cleanly when its tool is absent (`command -v`).
4. Run the checks below and make sure they pass.
5. Describe what you changed and why, and note anything a reviewer should verify by hand.

## Run the checks

No root and no radio are needed for these:

```bash
bash -n cp.sh                 # syntax
shellcheck cp.sh              # lint (install separately), keep 0 errors
for t in test_*.sh; do bash "$t"; done   # function + report harnesses
```

Every harness prints `RESULT pass=<n> fail=<n>` and exits non-zero on failure. Add or update a
harness when you change behavior, and keep the whole set green. See
[`docs/setup-and-testing.md`](docs/setup-and-testing.md).

## Conventions in this codebase

- **Shell style.** Functions do one job and have verb-first names. Preconditions are guard clauses at
  the top of a function that `print_log` and return, rather than deep nesting.
- **One output vocabulary.** All user-visible output goes through `print_log`. Console attack and
  result lines stay short (a few words). Detail belongs in the log and the proof file.
- **Severity is proof-gated.** A finding is Critical only when a key was recovered and a proof file
  exists for it. Posture findings stay Low/Info and must carry the measured beacon facts that back
  them. Do not raise a severity without evidence on disk.
- **Validate at the boundaries.** Values from the air or from the operator pass through
  `validate_env`, `normalize_bssid`, `json_escape`, or `_html_escape` before use.
- **Keep the defaults honest.** If you change a default that affects scope or disclosure, update
  [`docs/security.md`](docs/security.md) in the same change.

## License

By contributing you agree that your contribution is licensed under the MIT License, the same license
as the project. See [LICENSE](LICENSE).
