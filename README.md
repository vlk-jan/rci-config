# RCI Cluster Configuration

A collection of wrapper scripts for the [RCI cluster](https://login.rci.cvut.cz/) at CTU Prague. The cluster uses SLURM; this repo provides short, interactive commands for the common workflows (submitting jobs, opening Jupyter, tunneling, etc.) so you don't have to remember raw `sbatch` / `srun` / `squeue` invocations.

## Example

Instead of the raw SLURM flow:

```bash
$ squeue -u <username>
    JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
    7261076   gpufast     bash <username>  R       0:14      1 n21
$ scancel 7261076
```

You pick a job interactively:

```bash
$ cancel-job

Jobs for user <username>:
            JOBID PARTITION NAME USER ST TIME NODES NODELIST(REASON)
[1]       7261076   gpufast bash ...

Enter the number of the job to cancel: 1
Job 7261076 has been cancelled.
```

## Repository structure

```
.
├── commands/
│   ├── rci/           # Commands to run on the cluster (login / compute nodes)
│   └── local/         # Commands to run on your own machine
├── envs/
│   └── base.sh        # Default module set (used when a project has no .env / .venv)
├── jobs/              # Jinja2 templates for SLURM batch scripts
├── src/
│   └── create_job_file.py   # Renders the Jinja2 template for run-jupyter
├── configure_rci.sh   # One-time setup on the cluster
└── configure_local.sh # One-time setup on your machine
```

## Setup

### On RCI

Clone into `$HOME` (the path matters — scripts assume `~/rci-config`):

```bash
cd ~
git clone <repo-url> rci-config
bash rci-config/configure_rci.sh
```

This makes the commands executable and adds `~/rci-config/commands/rci` to your `PATH`. Start a new shell (or `source ~/.bashrc`) afterwards.

### Locally

```bash
git clone <repo-url> rci-config
bash rci-config/configure_local.sh
```

Prompts once for your RCI username, exports it as `$RCI_USER`, and adds `commands/local` to your `PATH`.

## Project layout on RCI

Put each project in `~/projects/<name>/`. The scripts look there and auto-detect the environment in this order:

1. `~/projects/<name>/.env` — a shell file that's `source`d (`ml …` calls, `source .venv/bin/activate`, etc.)
2. `~/projects/<name>/.venv/bin/activate` — a Python virtualenv (works with `uv`, `venv`, `virtualenv`)
3. Fallback — prompts for the base modules (`envs/base.sh`) or a Singularity image

Singularity images go in `/mnt/personal/<user>/singularity/<project>.sif`.

## Commands (on RCI)

| Command | What it does |
|---|---|
| `my-jobs` | Lists your queued / running SLURM jobs. |
| `job-history [days]` | Shows finished/failed/cancelled jobs (via `sacct`) with state, exit code, elapsed time, requested vs. peak memory, and GPU count. `my-jobs` only shows what's still queued; this is for after the fact. Defaults to the last 14 days. |
| `interactive-job` | Starts an interactive shell on a compute node. Auto-detects `.env` / `.venv` and activates it; picks from live availability; supports `any-gpu` / `any-cpu`. Remembers last settings. |
| `connect-job` | Attaches a shell to an already-running job (`srun --overlap`). |
| `cancel-job` | Cancels one of your jobs. |
| `job-logs` | Finds and `tail -f`'s the log file of a running job. |
| `run-jupyter` | Submits a Jupyter notebook as a SLURM batch job. Prints the SSH tunnel command to run locally. Remembers last settings. |
| `submit-job` | Submits an arbitrary command (e.g. a training script) as a SLURM batch job. Run from inside the project directory - unlike the commands above, it uses `$PWD` rather than `~/projects/<name>`. See [Submitting batch jobs](#submitting-batch-jobs). |

### Partition selection

`interactive-job`, `run-jupyter` and `submit-job` query `sinfo` at startup and show only partitions that allow interactive jobs (time limit ≤ 4h). Availability is shown in idle/total GPUs for GPU partitions, idle/total nodes for CPU partitions (down/draining nodes excluded from both). Two virtual options are added:

- `any-gpu` — picks whichever fast GPU partition has the most idle GPUs right now
- `any-cpu` — same for CPU partitions, ranked by idle nodes

GPU vs CPU is detected from each partition's GRES field, so new partitions added to the cluster appear automatically.

### Remembering settings

After your first run, `interactive-job`, `run-jupyter` and `submit-job` save your choices (partition, GPUs, memory, CPUs, ...) to `~/.interactive-job-last` / `~/.run-jupyter-last` / `~/.submit-job-last`. Subsequent runs offer to reuse them with a single Enter.

### Submitting batch jobs

`submit-job` targets the common case: run one command with N GPUs on some partition, no hand-written `.batch` file needed. It's modeled on the pattern of `--nodes 1 --ntasks-per-node 1`, split `--output`/`--error` logs, `source .env`, then a plain command:

```bash
$ cd ~/projects/my-project     # or wherever the project actually lives
$ submit-job
Command to run (e.g. python3 train.py --epochs 10): python3 train.py --dataset helhest --log_path ./logs/temporal
Job name [default=train]:
# ... partition / GPUs / memory / CPUs / time limit / email notifications ...
Submitted batch job 123456
```

It writes logs to `slurm/out/<job-name>-<job-id>.{out,err}` and keeps a permanent copy of the exact script it submitted in `slurm/generated/<job-name>-<job-id>.sh` (unlike `run-jupyter`, which deletes its generated script after submission - a one-off batch job is worth keeping a reproducible record of, an ephemeral Jupyter kernel isn't).

Jobs that need `--array` or multiple backgrounded processes in one allocation are still better off as a hand-written `.batch` file - `submit-job` doesn't try to template those.

## Commands (locally)

| Command | What it does |
|---|---|
| `ssh-rci` | Opens an SSH session to `login3.rci.cvut.cz` using `$RCI_USER`. |
| `rci-tunnel [node] [port]` | Opens an SSH tunnel from your machine to a compute node. Called with no args it prompts interactively. `run-jupyter` prints the exact invocation you can copy-paste. |

## Typical workflow: remote Jupyter

On RCI:

```bash
$ run-jupyter
# (select project, partition, resources — or reuse last)
# ... job starts ...
On your local machine, run:
    rci-tunnel n22 9999
```

Locally:

```bash
$ rci-tunnel n22 9999
```

Then open `http://localhost:9999` in your browser.

## Home directory layout

```
~/
├── .bashrc
├── rci-config/                # This repo — MUST be cloned here
├── projects/
│   ├── project-1/             # Uses RCI modules via .env
│   │   ├── .env
│   │   └── ...
│   ├── project-2/             # Uses uv / venv
│   │   ├── .venv/
│   │   ├── pyproject.toml
│   │   └── ...
│   └── project-3/             # Uses a Singularity container in /mnt/personal
│       └── ...
```

Personal storage (`/mnt/personal/<user>`) is used for datasets and Singularity images:

```
/mnt/personal/<user>/
└── singularity/
    ├── project-3.sif
    └── base.sif               # Optional fallback image
```
