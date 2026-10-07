# Lab HPC — Orientation & Operations Manual

**Admins:** Clifton Lewis, M. Eleonora Rossi, Niño Posadas

**Status:** Living document — the server was recently rebuilt and policies will keep evolving as people start using it. If something here is wrong, out of date, or you have a suggestion, tell me about so this doc can be updated. Bear in mind I'm not an IT admin I'm just learning all of this myself too.

Before using the lab server, please make sure you are comfortable using a linux system as we do not want to cause issues for other people and their data.

Ask for advice and suggestions if you aren't sure.

A good refernce book is ["Practical computing for biologists" by Haddock and Dunn (2011)](https://practicalcomputing.org/index.html).

Another good reference cheat sheet is [BASH Cheat Sheet](https://linuxstans.com/bash-cheat-sheet/).
**Last revised:** 2026-10-07

---

## 1. Overview

The lab HPC server ("evolution") was recently rebuilt from scratch. All drives were reformatted, so **nothing from before this rebuild still exists** — if you had data on the old system, it is gone unless it was backed up elsewhere.

The server has:
- 48 CPUs / ~1TB RAM (see partition limits in [Section 6](#6-slurm-job-scheduler))
- Two fast NVMe SSD drives (3.7TB each) for the OS, home directories, and scratch
- One larger 12TB HDD (slower, split into two 5.5TB volumes) for longer-term project storage
- Slurm for job scheduling and queueing
- Apptainer for running Docker/Singularity container images

---

## 2. Access, Login & Passwords

The server sits on the lab network at `IP_ADDRESS`. Once you're on that network, SSH in with the username your admin created for you:

```bash
ssh your_username@IP_ADDRESS
```

You'll be prompted for the password your admin set when your account was created. There's no self-service password reset — if you forget it, ask your admin to set a new one.

### If you've logged into this server before
The rebuild replaced the server's SSH host key, so your machine still has the *old* one cached. SSH will refuse to connect and warn that the "remote host identification has changed" — normally a sign something's impersonating the server, but here it's expected because of the rebuild. Clear the stale entry first:

```bash
ssh-keygen -R IP_ADDRESS
```

Then SSH in as above. Your machine will save the new host key and ask you to confirm it the first time — that's normal for any server you're connecting to for the first time.

Only admins can create new accounts — see [Section 4](#4-requesting--creating-a-new-account).

---

## 3. Storage Layout

The rebuild moved to a clear two-tier storage model: **fast but limited** (NVMe) for active work, and **slower but large** (HDD) for things that need to stick around.

| Drive | Type | Size | Purpose | Path |
|---|---|---|---|---|
| Drive 1 | NVMe SSD | 3.7TB | OS + home directories | `/home/<username>` |
| Drive 2 | NVMe SSD | 3.7TB | Scratch — active job I/O | `/scratch/<username>` |
| HDD partition 1 | HDD | 5.5TB | Long-term project storage | `/mnt/hpc_projects_1/<username>` |
| HDD partition 2 | HDD | 5.5TB | Long-term project storage | `/mnt/hpc_projects_2/<username>` |

### 3.1 Home (`/home/<username>`)
Lives on the fast NVMe drive alongside the OS. Use it for your conda environment, configs, scripts, and small personal files — not for large datasets or heavy read/write during a running job.

You can access your `home` directory like this:
```bash
cd /home/$USER
```
### 3.2 Scratch (`/scratch/<username>`)
This is where **active analysis should actually run**. It's on the fast NVMe drive, so it handles the heavy read/write that pipelines generate far better than the HDD storage does.

You can access the `scratch` directory like this:
```bash
cd /scratch/$USER
```

**Scratch is purged periodically.** It is *not* permanent storage — treat it as a fast workbench, not an archive. Move finished results to `/mnt/hpc_projects_1` or `/mnt/hpc_projects_2` once a run is done. Explained more below.

### 3.3 Long-term project storage (`/mnt/hpc_projects_1` and `/mnt/hpc_projects_2`)
Two 5.5TB partitions carved out of the single 12TB HDD. Slower than scratch, but this is where finished results, reference databases, and anything you need to keep for the longer term should live.

To access each of these storage directories, use either of the 2 commands below:
```bash
cd /mnt/hpc_projects_1/$USER
cd /mnt/hpc_projects_2/$USER
```

Shared reference data (e.g. BLAST databases) is also kept here so everyone can point their jobs at one copy instead of duplicating it — see the `BLAST_DB_DIR` example in the Slurm template ([Section 6.4](#64-submission-template-walkthrough)).

### 3.4 Shared scratch folder — `/scratch/shared`

There is a shared drop folder in scratch for passing files between people without emailing/copying things around manually.

**Permissions: `1777` (sticky bit + full read/write/execute for everyone)**

| Code | Meaning |
|---|---|
| `777` | Every user can open the folder, read, run, and create files in it |
| `1` (sticky bit) | Restricts **deletion** — only the file's owner (or an admin) can delete or rename it |

What this means in practice:

1. **Sharing files** — drop a script or file into `/scratch/shared` and anyone can instantly view, copy, or run it.
2. **Normal deletion** — if you created a file there, you can delete it yourself at any time, no admin needed.
3. **Protected from others** — if someone else tries to delete *your* file, Linux blocks it outright ("Permission Denied"), whether the attempt was accidental or not.
4. **Admin override** — if a file genuinely needs to be removed by someone other than its owner, ask the admin. They can use `sudo` to remove it.

Because this folder is shared and easy to dump things into, it is included in the periodic scratch purge — **don't use it as long-term storage**, only as a handoff point.

### **DO NOT RUN TASKS DIRECTLY IN THE SHARED FOLDER!**

---

## 4. Requesting / Creating a New Account

Only a small number of people have admin access, specifically so that new accounts get set up correctly and consistently across *every* drive (home, scratch, and both project volumes).

If you need an account, contact an admin directly — don't create one yourself even if you have sudo on your own machine.

---

## 5. Slurm Job Scheduler

All computational jobs should be submitted through Slurm rather than run directly on the login shell — the server is shared, and running heavy jobs outside Slurm starves everyone else.

Please use this cheat sheet as a good reference point. [SLURM_Cheat_Sheet](https://slurm.schedmd.com/pdfs/summary.pdf)

### 5.1 Partitions (queues)

Jobs are routed through partitions based on how long they'll run and how many cores/how much memory they need. All partitions run on the single node, `evolution`.

| Partition | Max walltime | Max CPUs | Max memory | Notes |
|---|---|---|---|---|
| `short` | 24:00:00 | 20 | 80GB | **Default** if you don't specify a partition |
| `short-multicore` | 24:00:00 | 40 | 200GB | Same time limit, more cores/memory |
| `medium` | 7 days | 20 | 500GB | |
| `medium-multicore` | 7 days | 40 | 600GB | |
| `long` | 2 months | 20 | 600GB | |
| `long-multicore` | 2 months | 40 | 850GB | Same cap, more cores/memory |
| `max` | Infinite | 48 | 950GB | **Requires admin approval** — restricted to `root` and the `hpc_max_users` group |

**Choosing a partition:** pick the smallest one that comfortably covers your job's expected runtime and resource needs. Reserving `medium`/`long` or a `multicore` tier for a job that finishes in an hour just blocks resources other people could be using.

### 5.2 How priority is decided (fair-share)

Slurm here uses the **multifactor priority** plugin, weighted heavily toward fair-share:

- **Fair-share weight is very high relative to age** — your priority is driven mostly by how much you've used the cluster *recently* compared to other users, not by how long your job has been waiting.
- **Recent usage decays over a 7-day half-life** — heavy usage from two weeks ago is essentially forgotten. Only roughly the last week of usage counts against your priority.
- Practical effect: if you've been running a lot of jobs lately, new jobs you submit will queue behind other users' jobs until your recent usage cools off. This resets on its own — you don't need to ask for anything.

### 5.3 Everyday Slurm commands

| Command | Purpose |
|---|---|
| `sbatch filename.sbatch` | Submit a job script |
| `squeue` | See queued/running jobs (add `-u $USER` for just yours) |
| `scancel <jobid>` | Cancel a job |
| `sinfo` | See partition/node status |
| `sacct -j <jobid>` | Check a finished job's resource usage (useful for right-sizing future requests) |

### 5.4 Submission template walkthrough

A ready-to-copy template lives at:

```bash
/scratch/shared/submit_template.sbatch
```

Copy it into your working directory, edit the fields below, and submit with `sbatch filename.sbatch`.

1. **Job name & logs** — `--job-name`, `--output`, `--error`. `%j` in a filename is replaced with the job's unique Slurm ID, so logs from different runs don't overwrite each other.
2. **Partition** — pick from the table in [5.1](#51-partitions-queues). Defaults to `short` if omitted.
3. **Time limit** — `--time=HH:MM:SS` or `DD-HH:MM:SS`. The job is killed if it hits this wall-clock limit, so pad it a bit, but don't wildly overshoot (it affects queuing/fair-share).
4. **CPUs & memory** — `--cpus-per-task` and `--mem`. **If you leave `--mem` blank, you're automatically defaulted to 200GB** — set it explicitly if your job needs less, so you're not holding memory hostage from other users.
5. **Email notifications (optional but heavily recommended)** — uncomment `--mail-type` and `--mail-user` to get an email when a job ends or fails.
6. **Pipeline body** — always `cd` into your scratch directory (`/scratch/$USER`) for the actual read/write work, and reference shared reference databases (e.g. BLAST DBs) from `/mnt/hpc_projects_1/databases/...` rather than copying them locally.

Minimal example:

```bash
#!/bin/bash
#SBATCH --job-name=my_analysis
#SBATCH --output=job_%j.out
#SBATCH --error=job_%j.err
#SBATCH --partition=short
#SBATCH --time=12:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G

WORK_DIR="/scratch/${USER}"
cd $WORK_DIR

blastn -query input.fasta -db /mnt/hpc_projects_1/databases/blast/nt \
  -num_threads ${SLURM_CPUS_PER_TASK} -out results.txt
```

---

## 6. Containers (Apptainer)

Apptainer is installed, so Docker and Singularity container images can be used directly without needing Docker itself (which typically isn't appropriate on a shared multi-user HPC node).

> ```bash
> apptainer pull docker://some/image:tag
> apptainer exec my_image.sif my_command --args
> ```

---

## 7. Good Rules on a Shared Server

- **Run jobs through Slurm, not the login node.** Heavy work directly on the shell slows the server down for everyone.
- **Do your heavy I/O in `/scratch/$USER`**, not on `/mnt/hpc_projects_*` — the HDD volumes are noticeably slower for active read/write.
- **Move results out of scratch once you're done** — it gets purged periodically (potentially without notice), and it's shared, so don't let it fill up with old runs.
- **Set `--mem` explicitly** rather than relying on the 200GB default — it's often more than your job needs and ties up memory other people could use.
- **Only use `max` if you've cleared it with an admin** — it's gated to `hpc_max_users` for a reason.
- **Don't delete other people's files in `/scratch/shared`** — you can't anyway (sticky bit), but if something there genuinely needs removing, ask an admin.

---

## 8. Getting Help / Giving Feedback

This system is new and still being tuned as real workloads hit it. If you:
- hit a wall you don't understand (permissions, quotas, job failures),
- think a partition limit, memory default, or purge policy should change, or
- have suggestions for how this manual or the setup could be improved, tell me please. Policies here (purge timing, partition limits, `max` access) are expected to be adjusted as actual usage patterns become clear.

:)
