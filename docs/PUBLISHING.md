# Publishing (first release)

> Manual steps for the account owner. CI handles every release AFTER
> these one-time steps. Tags: `py@0.2.0` and `ts@0.2.0` (matching the
> `0.2.0` version in `pyproject.toml` / `ts/package.json`; the `0.2.0`
> tree is what ships, so the tags name it — not `0.1.0`).

## Python (PyPI, package `tesera`)

1. Confirm MFA is enabled on the PyPI account (hardware key or TOTP,
   not SMS). Create the account at
   https://pypi.org/account/register/ first if needed.
2. Confirm Trusted Publishing is configured at
   `pypi.org/manage/account/publishing/`: pending publisher with owner
   `sheringfords`, repository `tesera`, workflow `publish-python.yml`,
   environment `pypi`.
3. From a clean checkout: `uv build`.
4. From the same checkout: `uvx twine check dist/*` (must pass).
5. Run the first publish: `uvx twine upload dist/*` using an
   account-scoped API token (not a project-scoped token, because the
   project does not exist yet; username `__token__`).
6. Confirm the package appears at `pypi.org/project/tesera/`.
7. Report back the exact version number.

## TypeScript (npm, package `tesera`)

1. Confirm MFA is enabled on the npm account (hardware key or TOTP,
   not SMS; `npm profile enable-2fa` — auth-and-writes level).
2. Run `pnpm --dir ts build`.
3. Run `pnpm --dir ts pack` and confirm the tarball contains only the
   intended files (32 files: `dist/src` js+d.ts, `verifiers/verify.mjs`,
   README, LICENSE, package.json).
4. Run the first publish: `cd ts && npm publish --access public`
   (the first publish is manual, not via Trusted Publishing, because
   the package does not exist yet; `--access public` is required).
5. Confirm the package appears at `npmjs.com/package/tesera`.
6. Report back the exact version number.

## After publish

1. Configure Trusted Publishing for the package on both registries
   (PyPI: `pypi.org/manage/account/publishing/`; npm: package Settings
   → Trusted Publisher), pointing at repo `sheringfords/tesera` and the
   two workflow files — so the NEXT release publishes from a tag.
2. Do NOT push `py@*`/`ts@*` tags for this release: the release
   workflows trigger on tags and would fail against the just-published
   version. Tag-driven publishing starts with the next version bump.
3. Report to the agent: "Published. Python \<version\>, TypeScript
   \<version\>." The agent will then run post-publish verification
   (fresh installs, README example, both verifiers, provenance) and
   write `docs/PUBLICATION_VERIFIED.md`.

## If a name is taken

Stop. Do not silently pick an alternative. The availability check in
`docs/NAMING_DECISION.md` was done pre-rename; re-check, propose, and
get written confirmation before touching a package name.
