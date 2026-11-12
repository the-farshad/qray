# slurmtop

A single-file Bash dashboard for your Slurm jobs, how efficiently they run, their priority
and queue wait, your usage, how busy the GPU partitions are, and your storage, refreshed live
in the terminal.

![slurmtop screenshot](screenshot.png)

## Features

Nine panels, each shown or hidden with its number key, like in `btop`:

| Key | Panel | Shown by default | What it shows |
|-----|-------|------------------|---------------|
| `1` | my jobs | yes | Your jobs, including job arrays: state, run time, time limit, CPUs, memory, GPUs, priority, expected start, and the nodes or the reason a job is waiting |
| `2` | running-job efficiency | no | For each running job: CPU use since it started, and memory and GPU use now, against what it was given |
| `3` | priority | yes | The priority breakdown of your pending jobs and the fair-share of your accounts |
| `4` | queue wait | yes | Median and longest time your jobs waited to start in the last 30 days, per partition |
| `5` | my usage | yes | CPU-hours, GPU-hours (also by GPU type) and jobs for the last hour, the last 7 days, this month, and this year; this month's jobs by final state; and how much of the allocated CPU time, the requested memory, and the time limit your finished jobs used, with the latest finished jobs listed (like `seff`) |
| `6` | failed jobs | no | This month's failed jobs grouped by cause (exit code, signal, time limit, memory, node failure), and the latest ones |
| `7` | GPU partitions | yes | GPUs used, down, and free, idle nodes, how many jobs are waiting (other users' jobs appear only as counts), and why nodes are down or draining |
| `8` | limits | no | Limits of your accounts, the time and size limits of the QoS they may use, and the time limit of each shown partition |
| `9` | storage | yes | Space used on the paths you choose |

Also:

- A warning line when a maintenance reservation is under way or starts within 14 days
- A daily log: one line of usage a day, for your own records
- Themes (`t`): default, nord, gruvbox, light (for light terminal backgrounds), and mono
- Compact mode (`c`): panels without frames and with fewer rows
- Layout (`l`): two columns in wide windows and one in narrow or vertical ones, or a fixed
  choice of one or two columns
- Runs on a cluster login node, or on your own computer over ssh, reusing one connection
- Light on the cluster (see below)
- Redraws in place without flicker; rounded panels and gradient meters in 256-color
  terminals, with fallbacks for basic and non-UTF-8 terminals

## Requirements

- On the machine that runs it: Bash 4+, `awk`, `tput`, and `ssh` when Slurm runs elsewhere
- On the cluster: the Slurm commands `squeue`, `sprio`, `sshare`, `scontrol`, `sinfo`,
  `sacct`, and `sacctmgr`, plus `awk`, `df`, GNU `date`, and `timeout`. The usage, queue
  wait, failed jobs, and limits panels need Slurm accounting (`slurmdbd`); without it they
  say so and the rest works as before
- For running-job efficiency: `srun` with `--overlap` (Slurm 20.11 or later), cgroup v2 on
  the compute nodes, and `nvidia-smi` for GPU figures
- For use over ssh: a login that needs no password prompt (an ssh key or agent)

## Install

```sh
curl -fsSLo slurmtop https://raw.githubusercontent.com/the-farshad/slurmtop/main/slurmtop
chmod +x slurmtop
./slurmtop
```

Or clone the repo and place the script on your `PATH`:

```sh
git clone https://github.com/the-farshad/slurmtop.git
cp slurmtop/slurmtop ~/.local/bin/
```

## Usage

```sh
slurmtop              # live view, refreshed every minute
slurmtop 30s          # refresh every 30 seconds
slurmtop 5m           # refresh every 5 minutes
slurmtop --once       # print one screen and exit
slurmtop --log        # add the daily log line if one is due, and exit
slurmtop -h           # show help
```

The interval can be 1 second to 1 hour, written as `30`, `30s`, `5m`, or `1h`. Shorter
intervals redraw more often, but Slurm itself is never asked more than once every
30 seconds.

### Keys

| Key | Action |
|-----|--------|
| `1`–`9` | Show or hide a panel |
| `t` | Next theme |
| `c` | Compact mode on or off |
| `l` | Layout: auto, one column, or two columns |
| `h` or `?` | Help: the keys, the panels and which are shown, and the file locations |
| `r` | Refresh now |
| `q` | Quit |

In the auto layout, panels are placed in two columns when the window is at least
181 columns wide. Each panel goes into whichever column is shorter, and the last panel of
the shorter column is stretched so both columns end on the same line. The wide tables
(panels 1, 2 and, outside compact mode, 5) keep the full width. Narrower or vertical windows
get one column.

Your panel, theme, compact, and layout choices are saved in
`~/.config/slurmtop/settings` (or `$XDG_CONFIG_HOME/slurmtop/settings`) and used the next
time.

### From your own computer

When Slurm is not installed locally, point `SLURMTOP_HOST` at a login node, either a host
name or a `Host` alias from `~/.ssh/config`:

```sh
SLURMTOP_HOST=login.mycluster.example slurmtop
```

The ssh connection is opened once and kept for 15 minutes, so later refreshes are quick.

### Daily log

Whenever slurmtop reads Slurm accounting and the last log line is 23 hours old or more, it
adds a line to `~/.local/state/slurmtop/daily.tsv` (or `$XDG_STATE_HOME/slurmtop/daily.tsv`).
The file is tab-separated with a header row, so it opens in a spreadsheet. Each line covers
the 24 hours before its time: CPU-hours, GPU-hours (also by GPU type), jobs that ran, jobs
that ended by state, how much of the allocated CPU time, requested memory, and time limit
those jobs used, and your fair-share and storage at that time.

Lines are written only while slurmtop runs. To log every day without it, run
`slurmtop --log` from cron, for example at 23:50:

```
50 23 * * * SLURMTOP_HOST=mycluster /path/to/slurmtop --log
```

## Light on the cluster

- Slurm is asked at most every 30 seconds, whatever the refresh interval; `r` and newly
  shown panels ask at most every 5 seconds
- Accounting, storage, maintenance reservations, limits, and the partition list are asked
  every 10 minutes, and the cluster name only at the start
- Hidden panels are not asked for: hiding panel 3 skips `sprio` and `sshare`, hiding
  panel 7 skips the node and pending-job queries, and hiding panel 8 skips `sacctmgr`
- All commands run on the login node at low priority (`nice`)
- Running-job efficiency is read every 5 minutes, and only while its panel is shown. For
  each running job (at most 16, 8 at a time) it starts a short step with
  `srun --overlap` that reads the job's cgroup counters and runs `nvidia-smi`, which takes
  well under a second. These steps appear in `sacct` as extra steps of your jobs.

## Configuration

All settings are environment variables:

| Variable              | Default                                     | Description                                                     |
|-----------------------|---------------------------------------------|-----------------------------------------------------------------|
| `SLURMTOP_HOST`       | none (Slurm on this machine)                | ssh host to run the Slurm commands on                           |
| `SLURMTOP_PARTITIONS` | GPU partitions your accounts and groups may use | Space-separated GPU partitions to show                       |
| `SLURMTOP_ACCOUNT`    | all of your accounts                        | Account to show the fair-share of                               |
| `SLURMTOP_PATHS`      | `~`                                         | Space-separated paths on the cluster to show storage for; `~` and `$USER` are expanded on the cluster |

Example, for a shell startup file:

```sh
export SLURMTOP_HOST=mycluster
export SLURMTOP_PARTITIONS="gpu-a100 gpu-h100"
export SLURMTOP_PATHS='~ /scratch/$USER'
```

## Notes

- GPU counts come from each node's configured and allocated GPUs (`gres/gpu`). A free GPU
  can still be unusable when its node has no free CPUs or memory.
- "jobs waiting" counts every user's pending jobs that may run in the partition; a job
  submitted to several partitions counts in each of them.
- The reasons for down nodes are the ones the administrators gave, without the name of the
  person who set them.
- Queue wait runs from when a job became eligible to start (submitted, and any dependency
  met) to when it started, so time spent waiting on your own dependencies or holds is left
  out. For job arrays, a task counts from when the array became eligible.
- Usage counts only the part of each job's run inside the period, and running jobs up to
  now. It is based on what jobs were allocated, not on how busy their CPUs or GPUs were.
- Efficiency of finished jobs compares what they used with what they asked for: CPU time
  with elapsed time × CPUs, peak memory (`MaxRSS`) with requested memory, and run time with
  the time limit. Values below 50% are highlighted. GPU utilization is not included.
- Running-job efficiency fills that gap for jobs that are still running: CPU time since the
  job started (from its cgroup), memory in use now (the cgroup's anonymous memory, without
  file cache), and GPU utilization and memory now (from `nvidia-smi`). For a job on several
  nodes, only its first node is read.
- Storage shows what `df` reports for each path: the size of the file system, or your
  quota on file systems that report quotas as their size.

## License

GNU General Public License v3.0 — see [LICENSE](LICENSE).
