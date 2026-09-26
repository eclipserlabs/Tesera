# systemd deployment for tesera witnesses

`tesera-witness.timer` (every 5 min) checkpoints the live journal and
ships the witness off-host. `tesera-verify.timer` (every 15 min) verifies
the chain, checks every shipped witness stays covered, and audits for
`needs_reconciliation` / `failed` invocations. Any non-zero exit should page.

```sh
sudo cp deploy/systemd/tesera-*.service deploy/systemd/tesera-*.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now tesera-witness.timer tesera-verify.timer \
  tesera-witness-prune.timer
systemctl list-timers 'tesera-*'
journalctl -u tesera-witness.service -u tesera-verify.service \
  -u tesera-witness-prune.service -f
```

`tesera-witness-prune.timer` runs monthly (`--keep 2000`, ~7 days of
depth at a 5-minute cadence); tune `TESERA_WITNESS_KEEP` to your
forensics window. Every `ExecStart` line is covered by
`tests/test_deploy_units.py`, which checks each subcommand and flag against
the real CLI parser — a typo'd flag fails CI, not a 3am timer.

Configure:

- `systemctl edit tesera-witness.service` to set `TESERA_JOURNAL`,
  `TESERA_WITNESS_DIR`, and optionally `TESERA_COUNTER_KEY`
  (`--counter-key` countersigns each witness with the external key).
- The witness dir must live where the journal cannot reach (separate mount,
  second host, WORM bucket). See `docs/DEPLOYMENT.md` for the S3 ObjectLock
  recipe and for pairing this with `WitnessFreshnessProvider`
  (`max_age_seconds` ≈ 2× the witness interval) so stale witnesses deny
  high-risk actions in-process instead of only paging afterwards.
- `OnFailure=` paging is yours: add `OnFailure=pager.service` to both
  `.service` units.
