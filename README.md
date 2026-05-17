# ProtForge

A SLURM-orchestrated pipeline for protein mutation scanning on the Kempner cluster.
Given a table of amino-acid mutations and a reference sequence, ProtForge generates
per-mutant FASTA inputs and runs up to four feature-generation stages end-to-end:

1. **MSA** — Multiple sequence alignment via local ColabFold / MMseqs2
2. **Boltz** — Structure prediction (`.cif`) with [Boltz](https://github.com/jwohlwend/boltz)
3. **ESM** — Per-residue embeddings from ESM-2
4. **ES** — Evolutionary-scale structural analysis via [PDAnalysis](https://github.com/mirabdi/PDAnalysis)

All four stages are wired together by a single config file and a single orchestrator
script (`run.sh`); each stage can be toggled on/off independently.

---

## Repository layout

```
.
├── config.yaml                 # Single source of truth for paths, resources, toggles
├── run.sh                      # Top-level orchestrator (submits all SLURM jobs)
├── download_tools.sh           # Installs/caches boltz, es, esm tooling
├── requirements-data.txt       # Python deps for the data-prep step
├── bash_scripts/
│   └── generate_data.sh        # Build per-mutant FASTA dir from a mutation table
├── slurm_scripts/              # Per-stage SLURM scripts + checkers
│   ├── parse_config.py         #   YAML reader used by all bash scripts
│   ├── split_and_run_msa.sh    #   Chunk inputs and submit MSA array
│   ├── split_and_run_boltz.sh  #   Chunk MSA outputs and submit Boltz array
│   ├── run_esm.sh / run_es.sh  #   ESM / ES launchers
│   ├── run_*_array.slrm        #   SLURM array job definitions
│   ├── checker.sh              #   Retry entry point (msa|boltz|esm)
│   └── checker_{msa,boltz,esm}.sh
└── utils/
    ├── generate_data.py        # Mutation table -> FASTA/YAML/cluster-FASTA
    ├── generate_subsamples.py  # Balanced / fixed-n-mutations subsampling
    └── utils.py                # Sequence I/O, mutation parsing, format converters
```

---

## Installation

1. Clone the repo:

   ```bash
   git clone git@github.com:dmbala/prot_mutationscan.git
   cd prot_mutationscan
   ```

2. Install the third-party tools you need. Pick any subset of `boltz`, `es`, `esm`:

   ```bash
   bash download_tools.sh --cache-dir /path/to/cache_base boltz es esm
   # or, reuse paths already defined in config.yaml:
   bash download_tools.sh --config config.yaml boltz es esm
   ```

   - `boltz`: clones and installs Boltz into `<cache>/boltz`; creates `<cache>/boltz_db`.
   - `es`: clones PDAnalysis into `<cache>/PDAnalysis` and installs it.
   - `esm`: prepares an ESM cache dir; if `fair-esm` is importable it pre-downloads
     `esm2_t33_650M_UR50D` weights.

   The relevant `*.cache_dir` / `*.script_dir` / `*.env_path` entries in `config.yaml`
   must point at these locations before you run the pipeline.

3. The data-prep step (`bash_scripts/generate_data.sh`) creates its own throwaway
   venv on first run (`.venv_data` in the project root) from `requirements-data.txt`
   — no manual Python setup needed.

---

## Step 1 — Prepare per-mutant input files

`bash_scripts/generate_data.sh` reads a TSV/CSV with an **`aaMutations`** column
(e.g. `SX123Y` or colon-separated multi-site `SA12V:SR45K`) plus a reference
sequence (`.fasta` or `.yaml`) and writes one file per mutant.

```bash
bash bash_scripts/generate_data.sh \
  --data      /path/to/mutations.tsv \
  --original  /path/to/reference.fasta \
  [--msa       /path/to/precomputed.a3m] \
  [--output_dir DIR]            # default: ./data/generated
  [--file_type cluster|fasta|yaml]   # default: cluster (one .fasta per mutant)
  [--subsample N]               # optional: subsample after generation
  [--subsample_mode balanced|fixed]
  [--num_mut N]                 # required for --subsample_mode fixed
  [--seed 42]
```

The script prints the absolute path to set as `input.fasta_dir` in `config.yaml`.

File-type choices:
- `cluster` (default) — one `seq_NNNNN.fasta` per mutant; this is what the
  cluster MSA stage consumes.
- `fasta` — `seq_NNNNN.fasta.txt` with an `>A|idx|msa` header.
- `yaml` — Boltz-style `seq_NNNNN.yaml` (skip directly to Boltz by setting
  `pipeline.msa: false` and `input.yaml_dir`).

---

## Step 2 — Configure `config.yaml`

The config is the single point of control. Key sections:

| Section | Purpose |
|---|---|
| `pipeline.{msa,boltz,esm,es}` | Toggle each stage (`true`/`false`) |
| `input.fasta_dir` | Directory of `.fasta` files produced in Step 1 (when MSA is on) |
| `input.yaml_dir` | Directory of pre-existing `.yaml` files (when MSA is off) |
| `output.parent_dir` | Where each run writes a `chunks_<timestamp>/` subdir |
| `msa.*` | Files-per-array-job, concurrency, MMseqs2 / ColabFold DB paths |
| `boltz.*` | Files-per-array-job, recycling steps, diffusion samples, cache, env |
| `esm.*` | Number of chunks, env path, work dir, model cache |
| `es.*` | PDAnalysis script dir, wild-type CIF, output dir, env |
| `slurm.{partition,account,log_dir}` | Cluster defaults |
| `slurm.<stage>.partition` | Optional per-stage partition override |

`run.sh` validates that the parameters needed by the **enabled** stages are present
before submitting anything, and it enforces:

- If `pipeline.msa: false`, then `input.yaml_dir` must be set.
- If `pipeline.es: true`, then `pipeline.boltz` must also be `true` (ES needs `.cif`s).

---

## Step 3 — Run the pipeline

```bash
./run.sh                       # uses ./config.yaml
./run.sh /path/to/other.yaml   # override config
```

`run.sh` submits the appropriate SLURM jobs and prints their IDs. Dependency chains
are set automatically:

- MSA runs first (or is skipped if `pipeline.msa: false` and `input.yaml_dir` is set).
- ESM and Boltz are submitted with `--dependency=afterok:<msa>`.
- Each main job has a paired *checker* submitted with `--dependency=afternotok:<main>`
  that runs only if the main job fails.
- ES is submitted last, after Boltz produces `.cif` files.

Outputs land in `<output.parent_dir>/chunks_<timestamp>/`.

---

## Retrying failures

If a stage fails (or partially fails), use the checker dispatcher to inspect
the run and re-submit only the missing pieces:

```bash
./slurm_scripts/checker.sh msa   /path/to/chunks_<timestamp> [config.yaml]
./slurm_scripts/checker.sh boltz /path/to/chunks_<timestamp> [config.yaml]
./slurm_scripts/checker.sh esm   /path/to/chunks_<timestamp> [config.yaml]
```

The checker re-exports config-derived environment (DB paths, env prefixes, partitions)
so retries use the same settings as the original run.

---

## Notes

- All paths in `config.yaml` should be absolute. `run.sh` normalises a few but does
  not edit the config.
- Stage-specific conda envs are activated via the `*.env_path` keys; nothing is
  installed into the system Python.
- The data-prep venv (`.venv_data`) is local to the repo and only used by
  `generate_data.sh`. Delete it to force re-creation.
- ColabFold MSA outputs are reused by Boltz; setting `boltz.delete_msa_after_processing:
  true` removes them after Boltz finishes successfully.
