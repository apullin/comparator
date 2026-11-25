# Harbor 1-vCPU Verifier Speedups (`core14–25` Macro-Pillars)

Measured from Harbor `[openverify-heartbeat]` logs across Phase 3 Macro-Pillar attempts (`#101612`, `#104525`, `#104848`, `#104989`, `#105775`). All three changes require **zero extra CPU cores** (safe on 1-vCPU container lanes) and save **~50–100 minutes per attempt**.

---

## 1. Precomputed Challenge Export (`--challenge-from-export`) — **Saves ~30–70 min / run**

Implemented in the parent commit (`Main.lean`, 100% backward-compatible with `comparator config.json`):

```bash
# Once per challenge image build (or shared read-only /data mount):
comparator config.json --save-challenge-export /data/challenge.export

# Per user submission (or set env COMPARATOR_CHALLENGE_EXPORT=/data/challenge.export):
comparator config.json --challenge-from-export /data/challenge.export
```

Skipping `safeLakeBuild challengeModule` + `safeExport challengeModule` avoids re-exporting the static `25M–114M`-token `Challenge` baseline on every user submission while keeping `Comparator.compareAt`, `Comparator.checkAxioms`, `nanoda`, and Lean's built-in kernel replay **100% identical**.

---

## 2. Emit Verdict Before `phase=cleanup` (or Skip Synchronous `rm -rf`) — **Saves ~6–12 min + Prevents False Timeouts**

On `#104848` (`core22`), `#104525` (`core20`), `#104989` (`core17`), and `#105775` (`core19`), `compile` and `compare` completed cleanly (`elapsed=5380s phase=compare`), and the container then hit `VERIFIER_TIMEOUT` (`exitCode 70`) **inside `phase=cleanup`** while `rm -rf`ing tens of thousands of hydrated CAS files prior to container teardown.

```python
# In the Harbor verifier entrypoint:
write_verdict_json(result_path, verdict)  # persist verdict immediately after `phase=compare`
# Skip synchronous `shutil.rmtree(work_dir)` when running inside an ephemeral container!
```

---

## 3. Hardlink / Symlink Read-Only CAS `.olean`s in `phase=hydrate_cas` — **Saves ~15–20 min (`~1,120s -> ~5s`)**

On `#101612` (`core18`), serial byte-copying in `phase=hydrate_cas` took `1,120s` (`18.7 min`) before `lbg exec` returned `502`. Because `/data/.openverify-lean-build-cache/v2-4331` is read-only and never mutated by `lake build Solution`:

```python
# Replace `shutil.copy2(cas_blob, dst)` in `hydrate_cas` with zero-copy link + fallback:
try:
    os.link(cas_blob, dst)
except OSError:
    os.symlink(cas_blob, dst)  # if `/data` and workspace are on different mounts
```
