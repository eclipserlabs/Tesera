# Publishing (first release)

> Manual steps for the account owner. CI handles every release AFTER
> these one-time steps. Tags: `py@0.2.0` and `ts@0.2.0` (matching the
> `0.2.0` version in `pyproject.toml` / `ts/package.json`; the `0.2.0`
> tree is what ships, so the tags name it — not `0.1.0`).

## Python (PyPI, package `tesera`)

1. Create a PyPI account at https://pypi.org/account/register/ and
   enable 2FA (account settings → two-factor).
2. On your machine: `uv build` (produces `dist/tesera-0.2.0-*.whl` and
   `dist/tesera-0.2.0.tar.gz`).
3. First upload (proves account control; creates the project):
   `uvx twine upload dist/*` — paste an account-scoped token
   (PyPI → API tokens → "entire account", username `__token__`).
4. Configure Trusted Publishing (PyPI → your profile → Publishing):
   add a pending publisher with owner `eclipserlabs`, repository
   `tesera`, workflow `publish-python.yml`, environment `pypi`.
5. Tag and push: `git tag py@0.2.0 && git push origin py@0.2.0`.
   The workflow builds and publishes via OIDC. No token in CI.
6. Confirm: `pip install tesera==0.2.0` in a fresh venv, run the README
   example, record in `docs/PUBLICATION_VERIFIED.md`.

## npm (package `tesera`)

1. Create an npm account at https://www.npmjs.com/signup and enable 2FA
   (`npm profile enable-2fa` — auth-and-writes level).
2. On your machine, in `ts/`: `pnpm install --frozen-lockfile &&
   pnpm build && pnpm pack` (produces `tesera-0.2.0.tgz`).
3. First publish (claims the name): `npm publish --access public`
   from `ts/` (logged in as the owner; `--access public` is required —
   the default for a new package would be restricted and fail).
4. Configure Trusted Publishing (npm → package Settings → Trusted
   Publisher): organization/user `eclipserlabs`, repository `tesera`,
   workflow `publish-npm.yml`. Requires npm CLI 11.5.1+ for
   `--provenance` (the workflow pins Node 20 + latest npm).
5. Tag and push: `git tag ts@0.2.0 && git push origin ts@0.2.0`.
   The workflow tests, builds, and publishes with `--provenance`
   via OIDC. No token in CI.
6. Confirm: `npm install tesera@0.2.0` in a fresh project, run the
   README example, record in `docs/PUBLICATION_VERIFIED.md`.

## If a name is taken

Stop. Do not silently pick an alternative. The availability check in
`docs/NAMING_DECISION.md` was done pre-rename; re-check, propose, and
get written confirmation before touching a package name.
