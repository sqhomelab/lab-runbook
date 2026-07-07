# Immich .env Corruption — Phantom Assets & Resource Crisis

**Date:** 2026-07-03 to 2026-07-05
**System(s) affected:** Immich (photo library, ~63GB), Postgres, Docker volumes
**Severity:** Data loss (recovered) / extended downtime

## Symptoms
- Immich showing phantom/missing assets despite the library appearing intact on disk
- Investigation into "why are assets missing" escalated into a resource crisis on the host
- A v2.7.5 → v3.0.0 schema migration subsequently wiped the database
- Real 63GB photo library was untouched on disk, but Immich behaved as if it wasn't there

## Root Cause
The `.env` file had stray `>` characters embedded in path values (likely from a shell redirect accidentally captured during editing). Docker interpreted the corrupted paths as valid but different locations, so containers mounted against empty decoy directories instead of the real library path. Immich saw "no assets" not because they were gone, but because it was looking in the wrong (empty) place the whole time. This silent mismatch is what triggered the phantom-asset investigation — the symptom (missing assets) had nothing to do with the actual data, only with where Docker was pointed.

## Investigation Path
- Initial assumption: assets were actually lost or corrupted → spent time checking library integrity, permissions, and Immich's asset DB records
- Resource crisis on host (RAM constrained to 3.3GB) surfaced during this investigation, adding noise and making it harder to isolate the real issue
- Eventually traced container mount points back to `.env` — found path values didn't match expected directories
- Diffing the `.env` against a known-good version revealed stray `>` characters silently altering path strings without throwing an obvious error

## Recovery Steps
1. Confirmed real library (63GB) was intact and untouched at its actual path
2. Corrected `.env` path values, removing stray `>` characters
3. Schema migration (v2.7.5 → v3.0.0) had already wiped the DB by this point — recovered via the built-in nightly SQL dump backup
4. Performed a full teardown and rebuild of Immich to v3.0.1
5. Applied resource caps on the rebuild to account for the 3.3GB RAM constraint on the host

## Prevention / Follow-up
- [ ] Rotate the exposed Postgres password
- [ ] Delete the exposed API key
- [ ] Remove `.bak` credential files
- [ ] Complete the three-way External Library split for per-user asset ownership
- [ ] Clean up orphaned directories (~53GB of old duplicate data)
- Consider a pre-flight `.env` validation check (e.g. reject any value containing `>`, `<`, `|`) before container restarts — would have caught this instantly instead of after a multi-day investigation
- Nightly SQL dump backup is what saved this — confirm it's still running post-rebuild and verify a restore periodically rather than assuming it works
