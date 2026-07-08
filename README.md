# InterSystems Programming Challenge 1: Code Golf

Minimal-code solution to the challenge: identify Gaia DR3 sources whose **BP or RP flux changed by more than 100%** over the observation period and write the result CSV. Optimized for **the smallest possible source** — both fewest lines *and* fewest characters (see the sibling repo `gaia-benchmark` for the speed-optimized, parallel variant).

The entire program is **one line** of Embedded Python — **524 characters** — living in a single ObjectScript class method. It reads the 20 gzipped input files, extracts and filters the flux arrays, computes the percentage change, and writes the sorted CSV, without ever leaving that one expression.

## Build & run

```bash
docker-compose up --build -d
docker-compose exec iris iris session iris
USER>do ^RunScript
```

This writes `data/out/challenge_output.csv` (≈10 seconds).

## How it works

`do ^RunScript` (`src/RunScript.mac`) calls `Gaia.Flux.Go()` (`src/Gaia/Flux.cls`, a single line of Embedded Python) which, in one pass:

- reads each `data/in/EpochPhotometry_*.csv.gz`,
- for the `bp_flux` and `rp_flux` arrays of every source, keeps only the valid fluxes and takes their `min` / `max`,
- computes `((max_flux - min_flux) / min_flux) * 100` per band and keeps the larger of the BP and RP values as `percentage_change`,
- writes the sources with `percentage_change > 100` (sorted descending) to the CSV.

Output columns: `source_id`, `bp_min_flux`, `bp_max_flux`, `rp_min_flux`, `rp_max_flux`, `percentage_change`.

The whole job is one `sorted(...)` expression fed to `csv.writer(...).writerows(...)`. A single shell pipe — `os.popen("zcat "+d+"in/*.gz|grep ^[0-9]")` — decompresses all 20 files as one stream and drops the comment and header lines (only data rows start with a digit), replacing the `glob`/`gzip` imports and the per-file loop. `eval(a,{"NaN":0})` parses each flux array without `json`, and a single `f>0` test keeps the valid values. Working in ratio space (`(max/min-1) > 1` ⇔ `> 100%`) lets the `×100` happen once at write time, and comprehension `for` clauses stand in for statements throughout. A pure-standard-library variant (no `zcat`/`grep`) is 561 characters; the shell pipe takes it to 524.

## Valid Flux Assumptions

The spec says to ignore *"missing, null, NaN, or otherwise invalid"* flux values. This solution treats a flux as valid only if it is **present, numeric, finite, and strictly positive** — the single `f>0` test drops `NaN` (which `eval` maps to `0`), zero, and negatives in one comparison, since `NaN>0`, `0>0`, and any negative are all `False`. (For these 20 files the arrays in fact contain only `NaN` and positive floats — no nulls, zeros, or negatives — so `f>0` is exactly sufficient.)

If a band has no valid fluxes, its `min`/`max` cells are left empty and only the other band contributes to `percentage_change`.

## Verifiable Result

- **3 lines of code total** — the `ROUTINE` header and one call in `RunScript.mac`, plus a single line in `Flux.cls`.
- **`Flux.cls` is 524 characters** (the Embedded Python method body is 463 characters; the rest is the class/method scaffolding), down from an initial 896.
