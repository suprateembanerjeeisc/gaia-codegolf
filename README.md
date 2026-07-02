# InterSystems Programming Challenge 1: Code Golf

Minimal-code solution to the challenge: identify Gaia DR3 sources whose **BP or RP flux changed by more than 100%** over the observation period and write the result CSV. Optimized for **fewest lines of code** (see the sibling release `gaia-benchmark` for the speed-optimized, parallel variant).

## Build & run

```bash
docker-compose up --build -d
docker-compose exec iris iris session iris
USER>do ^RunScript
```

This writes `data/out/challenge_output.csv` (≈10 seconds).

## How it works

`do ^RunScript` (`src/RunScript.mac`) calls `Gaia.Flux.Go()`
(`src/Gaia/Flux.cls`, a single line of Embedded Python) which, in one pass:

- reads each `data/in/EpochPhotometry_*.csv.gz`,
- for the `bp_flux` and `rp_flux` arrays of every source, keeps only the valid
  fluxes and takes their `min` / `max`,
- computes `((max_flux - min_flux) / min_flux) * 100` per band and keeps the
  larger of the BP and RP values as `percentage_change`,
- writes the sources with `percentage_change > 100` (sorted descending) to the
  CSV.

Output columns: `source_id`, `bp_min_flux`, `bp_max_flux`, `rp_min_flux`, `rp_max_flux`,
`percentage_change`.

## Valid Flux Assumptions

The spec says to ignore *"missing, null, NaN, or otherwise invalid"* flux values. This solution treats a flux as valid only if it is **present, numeric, finite, and strictly positive** — i.e. it drops:

- missing / `null` values
- `NaN` (detected via `flux == flux`, which is false only for NaN)
- zero and negative values

If a band has no valid fluxes, its `min`/`max` cells are left empty and only the
other band contributes to `percentage_change`.

## Verifiable Result

- **3 lines of code total** (`ROUTINE` header + one call in `RunScript.mac`; one line in `Flux.cls`).
