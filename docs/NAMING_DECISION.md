# Naming decision (publication, commit 0)

> Confirmed in writing by the human (Wira Mahendra) on 2026-09-26 via
> explicit selection: **"tesera everywhere"**.
> Exact spelling: `tesera` — T-E-S-E-R-A, one `s`. Never `tessera`.

## Confirmed names

| Artifact | Name |
|---|---|
| Product | Tesera |
| Python package (PyPI) | `tesera` |
| Python import | `tesera` (`import tesera`, `from tesera import guard`) |
| Python CLI | `tesera` (entry point `tesera = tesera.cli:main`) |
| TypeScript package (npm) | `tesera` |
| TypeScript imports | `tesera` (Effect modules under the same package) |
| Evidence home dir | `~/.tesera` (renamed; see CHANGELOG) |
| Env prefix | `TESERA_*` (renamed; see CHANGELOG) |
| Cloud repo/product | `tesera-cloud` / Tesera Cloud (private, unchanged spelling root) |

## Availability (checked 2026-09-26, before any rename)

| Registry | Name | Result |
|---|---|---|
| PyPI | `tesera` | 404 — available |
| PyPI | `tesera-evidence` | 404 — available (not chosen) |
| PyPI | `tesera-sdk` | 404 — available (not chosen) |
| npm | `tesera` | 404 — available |
| npm | `@eclipser/tesera` | 404 — available (not chosen) |

Method: `curl https://pypi.org/pypi/<name>/json` and
`https://registry.npmjs.org/<name>`; 404 means unclaimed. Scoped npm
check used the `%2f`-escaped form.

## Consequences

* `src/` subpackage renamed to `src/tesera/` (git mv). All internal
  imports renamed accordingly. No logic changes.
* The TypeScript package becomes `tesera` on npm.
* Pre-rename repository URLs and the colliding PyPI name are superseded
  (exact previous spellings recorded in CHANGELOG.md's rename entry);
  git repositories themselves are NOT renamed (per task constraints).
* The evidence format is untouched by the rename (bytes are identical;
  only the producing library's name changes).
