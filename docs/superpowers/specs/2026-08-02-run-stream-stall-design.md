# Control-plane run stream stall (sync dry-run)

## Symptom

Dashboard sync dry-run (`sync_verify`) appears stuck: live log stops mid-file-list, run meta stays `status: running`, cancel returns OK but does not finalize. Same command via CLI finishes in ~2s (`✓ Done`, ~25k files / 305G).

## Evidence

1. Orphan `rsync -avh -n …` (PPID 1) sampled stuck in `__write_nocancel` ← `vfprintf` ← stdout — **pipe buffer full**, writer blocked.
2. Run log mtime frozen (~11k lines) while meta still `running` / `finished_at: null`.
3. `POST /api/runs/cancel` returned `cancelled: true` but did not clear meta; bash parent gone, rsync survived.
4. CLI dry-run of identical argv completed successfully — `sync_to_backup.sh` is fine.

## Root cause (multi-bug)

### Bug 1 — Live stream backpressure stalls the drain loop (primary)

In [`web_browse.py`](scripts/web_browse.py) `_run_streaming`:

- Child stdout/stderr → reader threads → `queue` → **one handler thread** does `append_log` then `_emit_ndjson` (`wfile.write` + `flush`) per line.
- Sync uses `rsync -avh` → tens of thousands of path lines.
- Dashboard [`appendLiveLine`](outputs/dashboard.html) updates DOM on every NDJSON line (`textContent +=`, split, trim to 400, scroll).
- Slow browser consumer → TCP window fills → `_emit_ndjson` **blocks** → handler stops draining the queue → readers eventually stop consuming the pipe → **rsync blocks on write** → log and UI freeze.

CLI has no NDJSON/DOM backpressure, so it never hits this.

### Bug 2 — Cancel does not kill the process group

`Popen` without a new session; `terminate()` only signals bash. Children (`rsync`) can outlive the shell and keep the pipe open. Cancel also does not call `_finish_run_meta` itself — finalize only happens if `_run_streaming` unblocks, so a blocked emit leaves meta stuck on `running`.

### Bug 3 — Sync is too chatty for the control plane

`-avh` file lists are archive noise for the dashboard; they amplify Bug 1. Progress/stats would be enough.

## Design (fix)

### A. Decouple persist from live emit (must)

- Reader side (or a dedicated writer thread): **always** append to the run log with no dependency on the HTTP client.
- Emit to NDJSON with timeout / non-blocking / “drop if client slow”; never let client backpressure stop pipe drain.
- Prefer: log every line; live-emit sampled lines (e.g. first N, then every Kth, plus stderr and summary lines).

### B. Kill process group on cancel (must)

- `Popen(..., start_new_session=True)` (or explicit `preexec_fn=os.setsid`).
- Cancel: `os.killpg(proc.pid, SIGTERM)` then SIGKILL; then `_finish_run_meta(..., 'cancelled')` even if the stream thread is wedged (or unblock stream thread via closing pipes).

### C. Quieter sync for dashboard (should)

- Dashboard sync: drop `-v`, use `--info=stats2,progress2` (or equivalent) so stdout is progress/summary, not every path.
- Keep optional verbose behind a flag for CLI debugging.

### D. Frontend batching (should)

- Buffer NDJSON stdout/stderr and flush to DOM on rAF / 100–200ms, not per line.
- On stream stall, rely on existing log poll resume (already partially implemented).

## Acceptance

1. Sync dry-run from dashboard completes; meta ends `ok`/`error`/`cancelled` with `finished_at` set.
2. Cancel within 3s kills rsync tree; no orphan `rsync`; meta `cancelled`.
3. Live panel stays responsive; full path list remains in `_meta/logs/runs/*.log` if verbose CLI is used.
4. Regression test: fake command printing ≥20k lines through `_run_streaming` with a non-reading client does not deadlock; log reaches EOF and meta finalizes.
