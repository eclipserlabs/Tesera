# Pre-publish human tasks (under 20 minutes)

The agent cannot do these. Do them in order, then tell the agent
"Published. Python \<version\>, TypeScript \<version\>." after the manual
publish below.

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

## First manual publish (per docs/PUBLISHING.md)

- [ ] PyPI: `uv build` → `uvx twine upload dist/*` with an
      account-scoped token → confirm at `pypi.org/project/tesera/`.
- [ ] npm: `ts/` → `pnpm build && pnpm pack` (check files) →
      `npm publish --access public` → confirm at
      `npmjs.com/package/tesera`.
- [ ] Report versions back to the agent for post-publish verification.
