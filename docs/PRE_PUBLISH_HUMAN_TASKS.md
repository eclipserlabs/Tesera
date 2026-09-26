# Pre-publish human tasks (under 20 minutes)

The agent cannot do these. Do them in order: test first (per
docs/TEST_PUBLISH_SEQUENCE.md), then the real publish (per
docs/PUBLISHING.md Step 8). Tell the agent
"Published. Python \<version\>, TypeScript \<version\>." after the real
publish.

## TestPyPI prerequisites (before the test sequence)

- [ ] Create a TestPyPI account at
      https://test.pypi.org/account/register/ (separate from PyPI).
- [ ] Enable MFA on TestPyPI (hardware key or TOTP, not SMS).
- [ ] Generate a TestPyPI API token (account-scoped, because the
      project does not exist yet).
- [ ] Have the token ready before starting Step 2 of
      docs/TEST_PUBLISH_SEQUENCE.md (username is `__token__`).

## Security (from the audit closeout)

- [ ] Private vulnerability reporting: repo Settings → Security →
      "Private vulnerability reporting" → Enable. (Agent-checked: API
      returns no confirmed status; only a repo admin can set this.)
- [ ] Secret scanning: same page → enable push protection if offered.
- [ ] MFA (hardware key or TOTP, not SMS) on the PyPI account.
- [ ] MFA (hardware key or TOTP, not SMS) on the npm account.
- [ ] Confirm the SECURITY.md address (personal, best-effort,
      single-engineer) is monitored, or set up a dedicated alias first.
- [ ] Trusted Publishing on PyPI: `pypi.org/manage/account/publishing/`
      → add pending publisher for repo `sheringfords/tesera`, workflow
      `publish-python.yml`, environment `pypi`.
- [ ] Trusted Publishing on npm: package Settings → Trusted Publisher
      for repo `sheringfords/tesera`, workflow `publish-npm.yml`.

## Test first, then publish

- [ ] Run docs/TEST_PUBLISH_SEQUENCE.md end to end (TestPyPI upload,
      fresh-venv install, README example, npm dry-run + tarball
      install). Record the Step 1 SHA-256 values.
- [ ] If the test reveals a defect, report it to the agent and wait
      for a fix. Rebuild, retest.

## First manual publish (per docs/PUBLISHING.md Step 8, only after the test is clean)

- [ ] Rebuild from a clean worktree; confirm the SHA-256 matches the
      tested build. If it differs, stop and investigate.
- [ ] PyPI: `python3 -m build` → `python3 -m twine check dist/*` →
      `python3 -m twine upload dist/*` with the real PyPI
      account-scoped token → confirm at `pypi.org/project/tesera/`.
- [ ] npm: `ts/` → `pnpm build && npm pack --dry-run` (check files) →
      `npm publish --access public` → confirm at
      `npmjs.com/package/tesera`.
- [ ] Report versions back to the agent for post-publish verification.
