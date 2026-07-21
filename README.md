# InterSystems Programming Challenge 1: Code Golf

Minimal-code solution to the challenge: identify Gaia DR3 sources whose **BP or RP flux changed by more than 100%** over the observation period and write the result CSV. Optimized for **the smallest possible source** (see the sibling repo `gaia-benchmark` for the speed-optimized, parallel variant).

The entire program is **one line** of Embedded Python — **401 characters** — inlined directly in `RunScript.mac` via `$SYSTEM.Python.Run(...)`. It reads the 20 gzipped input files, extracts and filters the flux arrays, computes the percentage change, and writes the CSV, without ever leaving that one expression.

## Build & run

```bash
docker-compose up --build -d
docker-compose exec iris iris session iris
USER>do ^RunScript
```

This writes `data/out/r.csv`.

## How it works

`do ^RunScript` (`src/RunScript.mac`) runs a single line of Embedded Python which, in one pass:

- reads each `data/in/EpochPhotometry_*.csv.gz`,
- for the `bp_flux` and `rp_flux` arrays of every source, keeps only the valid fluxes and takes their `min` / `max`,
- computes `((max_flux - min_flux) / min_flux) * 100` per band and keeps the larger of the BP and RP values as `percentage_change`,
- writes the sources with `percentage_change > 100` to the CSV.

Output columns: `source_id`, `bp_min_flux`, `bp_max_flux`, `rp_min_flux`, `rp_max_flux`, `percentage_change`.

The whole job is one comprehension fed to `csv.writer(...).writerows(...)`. A single shell pipe — `os.popen('zcat /i/*|grep ^[0-9]')` — decompresses all 20 files as one stream and drops the comment and header lines (only data rows start with a digit), replacing the `glob`/`gzip` imports and the per-file loop. `eval(a,{'NaN':0})` parses each flux array without `json`, and a single `f>0` test keeps the valid values. Working in ratio space (`(max/min-1) > 1` ⇔ `> 100%`) lets the `×100` happen once at write time, and comprehension `for` clauses stand in for statements throughout.

### Character-count techniques

- **Inlined in `RunScript.mac`** via `$SYSTEM.Python.Run("...")` — no separate class file, so the counted unit is exactly the routine line the judge runs.
- **Single-quoted** Python string literals throughout, so no ObjectScript `"` → `""` escaping is needed inside `Run("...")`.
- **Short path aliases** — `docker-compose.yml` mounts `./data/in` at `/i` (read-only) and `./data/out` at `/o`, so the code uses `/i/*` and `/o/r.csv` instead of the long `/home/irisowner/dev/data/...` paths.

## Valid Flux Assumptions

The spec says to ignore *"missing, null, NaN, or otherwise invalid"* flux values. This solution treats a flux as valid only if it is **present, numeric, finite, and strictly positive** — the single `f>0` test drops `NaN` (which `eval` maps to `0`), zero, and negatives in one comparison, since `NaN>0`, `0>0`, and any negative are all `False`. (For these 20 files the arrays in fact contain only `NaN` and positive floats — no nulls, zeros, or negatives — so `f>0` is exactly sufficient.)

If a band has no valid fluxes, its `min`/`max` cells are `0` and only the other band contributes to `percentage_change`.

## Verifiable Result

- **The solution is 401 characters** — the single `$SYSTEM.Python.Run(...)` statement in `RunScript.mac` (the whole file, including the `ROUTINE RunScript` header, is 420).
- Produces `data/out/r.csv` with **57,099** qualifying sources.
