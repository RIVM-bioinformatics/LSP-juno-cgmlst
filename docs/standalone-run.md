# Running Juno-cgMLST standalone (outside iRODS)

`run_pipeline.sh` is built for the iRODS/LSF context: it hardcodes the
`/mnt/miniconda` conda install, maps iRODS project names to a genus via a
fixed `case` statement, always builds a `bsub` cluster command, and
propagates iRODS metadata env vars. None of that is needed to run the
pipeline standalone — [`juno_cgmlst.py`](../juno_cgmlst.py) can be invoked
directly.

## 1. Create the conda environment

```bash
mamba env create -f envs/juno_cgmlst.yaml --name juno_cgmlst
conda activate juno_cgmlst
```

## 2. Prepare inputs

- **Input directory**: a folder of `.fasta` assemblies (not raw reads —
  `input_type = "fasta"` in `juno_cgmlst.py`).
- **Genus** (`-g`): one of `campylobacter`, `escherichia`, `listeria`,
  `salmonella`, `shigella`, `yersinia`, `clostridioides`, `stec` (see
  [`files/dictionary_correct_cgmlst_scheme.yaml`](../files/dictionary_correct_cgmlst_scheme.yaml)).
- **DB directory** (`-d`): any writable local path. Missing cgMLST schemes
  are downloaded automatically by `download_missing_schemes()` in
  `juno_cgmlst.py`.

## 3. Run

> **Note:** `-d` can be omitted if the machine already has a populated
> `/mnt/db/juno/cgmlst` (the default from `juno_cgmlst.py`) with the
> scheme for your chosen genus already prepared under
> `prepared_schemes/<genus>` — this is the case on RIVM machines, e.g.
> `prepared_schemes/salmonella` is already present. That directory is
> often not writable by your user though, so it only works as long as
> every genus you request is already prepared there; if a genus is
> missing, the scheme download will fail on the permission error and
> you'll need to pass `-d <writable_dir>` instead.

```bash
python juno_cgmlst.py \
    -i <input_dir_with_fasta_assemblies> \
    -o <output_dir> \
    -g <genus> \
    -d <local_db_dir> \
    --local
```

The `--local` flag (from `juno_library.Pipeline`) tells snakemake to run
jobs on the local machine instead of submitting to an LSF cluster via
`bsub`.

### Worked example

Given an input directory of assemblies, put the output in a sibling
`cgmlst` directory (parallel to `assembly`), reusing the same run
subfolder name, and rely on the default `/mnt/db/juno/cgmlst` (no `-d`
needed since `prepared_schemes/salmonella` already exists there):

```bash
PROJECT_DIR=/data/BioGrid/gremmenr/projects/260707_cgmlst_v3_rerun
RUN_ID=260306_VH01799_389_AAHLGWYM5_0024

python juno_cgmlst.py \
    -i "$PROJECT_DIR/assembly/$RUN_ID" \
    -o "$PROJECT_DIR/cgmlst/$RUN_ID" \
    -g salmonella \
    --local
```

Logs land under `<output_dir>/log/...` per the `OUT` variable in the
`Snakefile`, e.g. for this example:

- `$PROJECT_DIR/cgmlst/$RUN_ID/log/cgmlst/list_samples_per_cgmlst_scheme.log`
- `$PROJECT_DIR/cgmlst/$RUN_ID/log/cgmlst/chewbbaca_salmonella.log`

Plus a run summary at `<output_dir>/audit_trail/snakemake_report.html`.

## Running standalone but still on the HPC (bsub)

Drop `--local` and pass `-q`/`-tl` instead:

```bash
python juno_cgmlst.py \
    -i <input_dir_with_fasta_assemblies> \
    -o <output_dir> \
    -g <genus> \
    -d <local_db_dir> \
    -q <queue_name> \
    -tl <time_limit_minutes>   # optional, default 60
```

Requires running from a node with `bsub` on `PATH`. Everything else
(conda env, genus values, DB dir) is unchanged.

### Worked example (HPC)

Same layout as above, but submitted to the `bio` queue instead of
running locally:

```bash
PROJECT_DIR=/data/BioGrid/gremmenr/projects/260707_cgmlst_v3_rerun
RUN_ID=260306_VH01799_389_AAHLGWYM5_0024

python juno_cgmlst.py \
    -i "$PROJECT_DIR/assembly/$RUN_ID" \
    -o "$PROJECT_DIR/cgmlst/$RUN_ID" \
    -g salmonella \
    -q bio
```

The same per-rule logs as the local example land under
`<output_dir>/log/cgmlst/`. Additionally, each `bsub` job's own
stdout/stderr goes to `<output_dir>/log/cluster/`, e.g.
`$PROJECT_DIR/cgmlst/$RUN_ID/log/cluster/<jobname>_<jobid>.out`
(built from `cluster_log_dir` in `juno_library.py`).

## Debugging against a local `juno-library` checkout

The env file's `pip:` section installs `juno_library` editable from
GitHub's `main` branch — but `pip` clones that into its own copy inside
the env (e.g. under `$CONDA_PREFIX/src/juno-library`), which is *not*
the same as an existing local clone. Edits made in your own working
checkout won't be picked up unless the env is pointed at it directly.

To debug with your own local clone (e.g. at
`/home/gremmenr/source/juno-library`), edit
`envs/juno_cgmlst.yaml`:

```yaml
- pip:
  - "--editable=/home/gremmenr/source/juno-library"
```

then recreate the env. Changes made in that checkout will now be picked
up immediately (no reinstall needed) since it's an editable install.

Prefer debugging with `--local` rather than on the HPC: with `--local`,
snakemake runs rules directly in your current shell, so you get
immediate stdout/stderr and can drop breakpoints into the orchestration
code. On the HPC path, each rule is shipped off via `bsub` to a separate
node with its own log files under `<output_dir>/log/cluster`, so you end
up queuing jobs and tailing remote logs instead of watching things run —
a much slower debug loop, even though the same Python code executes
either way.

## Notes

- Cluster-only options from the wrapper (`--queue`, LSF time limits) are
  irrelevant when running with `--local`.
