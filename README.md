# R + Python Singularity Container

Run R and Python side by side for data analysis using a pre-built Singularity/Apptainer image,
with **Jupyter Notebook** and **Claude Science** served from an HPC compute node to your local
browser.

The image ships R 4.5.0 and Python 3.10 on Ubuntu 22.04. Packages are *not* baked in: Python
packages live in a `venv` and R packages in an `renv` library, both on shared storage, so they
can change without rebuilding the image.

> Singularity was renamed Apptainer. On NEMO the command is `singularity`; `apptainer` is a
> drop-in equivalent elsewhere.

## Contents

- [Quick reference](#quick-reference)
- [1. One-time setup](#1-one-time-setup)
- [2. Start a session on NEMO](#2-start-a-session-on-nemo)
- [3. Start Jupyter Notebook (port 8888)](#3-start-jupyter-notebook-port-8888)
- [4. Start Claude Science (port 8889)](#4-start-claude-science-port-8889)
- [5. Shut down cleanly](#5-shut-down-cleanly)
- [Using the container outside NEMO](#using-the-container-outside-nemo)
- [Troubleshooting](#troubleshooting)
- [Appendix A: Building a new container](#appendix-a-building-a-new-container)
- [Links](#links)

---

## Quick reference

Two browser apps, two ports, two screen windows — everything else is identical between them.

| # | Where | Command |
|---|---|---|
| 1 | laptop | `ssh -L 8888:localhost:8888 -L 8889:localhost:8889 <user>@login007.nemo.thecrick.org` |
| 2 | login node | `screen`, then `Ctrl-a` `c` for a new window |
| 3 | login node (window 1) | `srun ... --pty bash`, then `squeue -u <user>` to read `<node_id>` |
| 4 | login node (windows 2 & 3) | `ssh -i ~/.ssh/id_rsa -L 8888:localhost:8888 <user>@<node_id>` — and the same with `8889` |
| 5 | compute node (both) | `ml Singularity/3.6.4` → `singularity shell ...` → `source ../envs/demo_venv_310/bin/activate` → `export RENV_PATHS_*` |
| 6 | container (window 2) | `jupyter notebook --no-browser --port=8888 --ip=127.0.0.1` |
| 7 | container (window 3) | `claude-science serve --no-browser --dangerously-no-sandbox --port 8889` |

Then in your **local** browser: the tokenised `http://127.0.0.1:8888/...` URL Jupyter prints,
and `http://127.0.0.1:8889` for Claude Science.

### Paths used in this guide

Substitute your own where they differ:

```bash
# Institute image (no download needed)
SIF=/flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif

# Your project root, laid out as in step 1.1
PROJECT=/nemo/lab/rouhanif/home/users/boeings/projects/demo

# Python venv, created inside the container (step 1.4)
VENV=$PROJECT/envs/demo_venv_310
```

All relative paths below (`../envs/...`) assume you are in **`$PROJECT/scripts`**.

---

## 1. One-time setup

Do this once per project. If someone has already set the project up, skip to
[step 2](#2-start-a-session-on-nemo).

### 1.1 Folder structure

```
demo/
├── scripts/
├── envs/
├── workdir/
└── data/
```

### 1.2 Get the container image

**Option A — use the institute path (recommended, no download):**

```
/flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif
```

**Option B — pull from DockerHub:**

```bash
cd envs
ml Singularity/3.6.4
singularity pull docker://boeings/r450.python310.ubuntu.22.04
```

This writes `../envs/r450.python310.ubuntu.22.04_latest.sif` — use that path wherever `$SIF`
appears below.

### 1.3 Clone the scripts

```bash
cd ../scripts
git clone git@github.com:decusInLabore/singularityInstance.git
```

### 1.4 Create the Python venv

Create it **from inside the container**, so it links against the container's Python 3.10:

```bash
ml Singularity/3.6.4
singularity shell --bind /nemo:/nemo,/camp:/camp,/flask:/flask \
  /flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif

cd $PROJECT/scripts
python3.10 -m venv ../envs/demo_venv_310
source ../envs/demo_venv_310/bin/activate

pip install jupyter notebook jupyterlab
# or restore a known-good set:
pip install -r venv.lock
```

### 1.5 Move the Claude Science data directory off your home quota

Claude Science writes its state to `~/.claude-science`, which will exhaust a small home
quota. Point it at scratch **before** the first run:

```bash
# 1. Create the real directory on large storage
mkdir -p /scratch/$USER/claude-science-data

# 2. If ~/.claude-science already exists with data in it, move it over first
if [ -d ~/.claude-science ] && [ ! -L ~/.claude-science ]; then
    mv ~/.claude-science/* /scratch/$USER/claude-science-data/ 2>/dev/null
    rmdir ~/.claude-science
fi

# 3. Symlink it
ln -s /scratch/$USER/claude-science-data ~/.claude-science
```

### 1.6 Install the R kernel for Jupyter

Once per new project, from an R session inside the container:

```r
renv::install('IRkernel')
renv::snapshot()
```

---

## 2. Start a session on NEMO

Steps 2.1–2.3 are done once per session. Step 2.4 is the environment block you will run in
**each** of the two windows that serve Jupyter and Claude Science.

### 2.1 Sign in to a login node

From your laptop or desktop, forwarding both ports:

```bash
ssh -L 8888:localhost:8888 -L 8889:localhost:8889 boeings@login007.nemo.thecrick.org
```

or, if your group has dedicated nodes:

```bash
ssh -L 8888:localhost:8888 -L 8889:localhost:8889 boeings@babs003.nemo.thecrick.org
```

> Connect to a **specific, numbered** login node (`login007`, not the round-robin alias), so
> that your later hops and tunnels all start from the same host.

### 2.2 Start a screen session

```bash
screen
```

| Action | Keys / command |
|---|---|
| New screen window | `Ctrl-a` then `c` |
| Switch between windows | `Ctrl-a` then `"` |
| Detach from session | `Ctrl-a` then `d` |
| Reconnect later | `screen -d -r` |

You will use three windows: **window 1** holds the interactive job, **window 2** serves Jupyter
on 8888, **window 3** serves Claude Science on 8889.

### 2.3 Claim an interactive compute node (window 1)

**CPU:**

```bash
srun --ntasks=1 --cpus-per-task=8 --partition=nint --time=08:00:00 --mem=256G --pty bash
```

**GPU** — only when your analysis actually needs one:

```bash
srun --job-name=G1 --ntasks=1 --time=16:00:00 --partition=vis --nodes=1 --gres=gpu:1 --pty --mem=200G bash
```

Wait for the node to start, then read its ID from the `NODELIST` column:

```bash
squeue -u <your_username>
```

That value is your `<node_id>` (e.g. `gl410`). **Leave window 1 open** — closing it releases
the allocation and kills both servers.

### 2.4 Tunnel in and enter the container

Do this in **window 2** (port 8888). You will repeat it in window 3 with port 8889 in
[step 4](#4-start-claude-science-port-8889).

Open a new screen window with `Ctrl-a` `c`, then hop from the login node to your compute node:

```bash
ssh -i ~/.ssh/id_rsa -L 8888:localhost:8888 <username>@<node_id>
# Crick example:
ssh -i ~/.ssh/id_rsa -L 8888:localhost:8888 boeings@gl410
```

> The **private** key is expected at `~/.ssh/id_rsa` — adjust the `-i` path if yours differs.
> The matching public key must be in `~/.ssh/authorized_keys` on NEMO for this hop to work
> without a password.

Load Singularity and start the container from your scripts directory:

```bash
cd $PROJECT/scripts
ml Singularity/3.6.4
```

**Option A — CPU:**

```bash
singularity shell --bind /nemo:/nemo,/camp:/camp,/flask:/flask \
  /flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif
```

**Option B — GPU** (`--nv` exposes the GPU; use only on a GPU allocation):

```bash
singularity shell --nv --bind /nemo:/nemo,/camp:/camp,/flask:/flask \
  /flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif
```

Activate the venv:

```bash
source ../envs/demo_venv_310/bin/activate
```

Point R and Python at the shared package caches:

```bash
export R_LIBS_USER=/nemo/lab/rouhanif/home/users/boeings/R/library/
export RENV_PATHS_LIBRARY=/nemo/stp/babs/working/boeings/package_caches/R/library/
export RENV_PATHS_CACHE=/nemo/stp/babs/working/boeings/package_caches/renv/cache/
export RENV_PATHS_ROOT=/nemo/stp/babs/working/boeings/package_caches/renv/
```

The environment is ready. You can now run `python` or `R` interactively, or continue to
step 3.

---

## 3. Start Jupyter Notebook (port 8888)

In window 2, inside the container:

```bash
jupyter notebook --no-browser --port=8888 --ip=127.0.0.1
```

Copy the URL printed in the terminal into your local browser, e.g.

```
http://127.0.0.1:8888/tree?token=6a671b92eac9d09f28971964ba1751147007844d2a688817
```

Open `singularityInstance/example_python_R_notebooks/Seurat_to_anndata_conversion.ipynb` and
**save it under a new name before editing**, so the example stays clean.

---

## 4. Start Claude Science (port 8889)

Repeat [step 2.4](#24-tunnel-in-and-enter-the-container) in **window 3** — everything is
identical except the port:

```bash
ssh -i ~/.ssh/id_rsa -L 8889:localhost:8889 <username>@<node_id>
# Crick example:
ssh -i ~/.ssh/id_rsa -L 8889:localhost:8889 boeings@gl410
```

<details>
<summary>Full copy-paste block for window 3</summary>

```bash
# from the login node, in a fresh screen window (Ctrl-a c)
ssh -i ~/.ssh/id_rsa -L 8889:localhost:8889 boeings@gl410

cd $PROJECT/scripts
ml Singularity/3.6.4

# add --nv if you are on a GPU allocation
singularity shell --bind /nemo:/nemo,/camp:/camp,/flask:/flask \
  /flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif

source ../envs/demo_venv_310/bin/activate

export R_LIBS_USER=/nemo/lab/rouhanif/home/users/boeings/R/library/
export RENV_PATHS_LIBRARY=/nemo/stp/babs/working/boeings/package_caches/R/library/
export RENV_PATHS_CACHE=/nemo/stp/babs/working/boeings/package_caches/renv/cache/
export RENV_PATHS_ROOT=/nemo/stp/babs/working/boeings/package_caches/renv/
```

</details>

Then start the server:

```bash
claude-science serve --no-browser --dangerously-no-sandbox --port 8889
```

Open the printed address in your local browser:

```
http://127.0.0.1:8889
```

> `--dangerously-no-sandbox` is required because you are already inside a Singularity
> container, which lacks the privileges the sandbox needs. Keep the flag to this context.

---

## 5. Shut down cleanly

1. Save your notebook.
2. Close the browser tabs — leaving them open produces repeated `Connection refused` messages
   in the terminal.
3. `Ctrl-C` in window 2 (Jupyter) and window 3 (Claude Science).
4. `exit` out of each container shell and each compute-node SSH hop.
5. In window 1, `exit` to release the allocation — or detach with `Ctrl-a` `d` to keep the node
   and come back later with `screen -d -r`.

---

## Using the container outside NEMO

The image is self-contained, so the same workflow runs on any host with Singularity/Apptainer.
Bind whatever directories you need instead of the Crick shares:

```bash
singularity shell --bind <local_dir_1>:/local_dir_1,<local_dir_2>:/local_dir_2 \
  ../envs/r450.python310.ubuntu.22.04_latest.sif
```

> Add `--cleanenv` if you hit environment-variable conflicts with the host.

Venv handling is the same as in [step 1.4](#14-create-the-python-venv):

```bash
python3.10 -m venv ../envs/demo_venv_310     # create
source ../envs/demo_venv_310/bin/activate    # activate
pip install <package>                        # install
pip install -r venv.lock                     # or restore from the lock file
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `jupyter: command not found` / won't start | The venv is not active, or Jupyter is missing: `pip install jupyter notebook jupyterlab` |
| Repeated `Connection refused` in the terminal | A browser tab is still attached to a previous Jupyter session — close it. |
| `Address already in use` on 8888/8889 | A previous tunnel or server still holds the port: `lsof -ti:8888 \| xargs kill -9`, then reconnect. Check the same port on your laptop. |
| Browser cannot connect at all | A hop is missing. Each port needs laptop → login node ([2.1](#21-sign-in-to-a-login-node)) **and** login node → compute node ([2.4](#24-tunnel-in-and-enter-the-container)). |
| `Permission denied (publickey)` on the login → node hop | Public key not in `~/.ssh/authorized_keys` on NEMO, or wrong `-i` path. |
| Node ID unknown, or servers died | `squeue -u <your_username>`. If nothing is listed, the allocation ended — redo [step 2.3](#23-claim-an-interactive-compute-node-window-1). |
| No R kernel in Jupyter | Install `IRkernel` as in [step 1.6](#16-install-the-r-kernel-for-jupyter). |
| R cannot find packages | The `export R_LIBS_USER` / `RENV_PATHS_*` block was not run in *this* shell. |
| Home quota full, or Claude Science fails to start | Move `~/.claude-science` to scratch as in [step 1.5](#15-move-the-claude-science-data-directory-off-your-home-quota). |
| GPU not visible (`nvidia-smi` fails) | Container started without `--nv`, or the job is on a CPU partition. |

---

## Appendix A: Building a new container

Only needed if you have to add system-level software. Day-to-day work uses the prebuilt image
from [step 1.2](#12-get-the-container-image).

Building requires **root**, which you do not have on the cluster. On Linux with `sudo`, skip
to [A.3](#a3-build-the-container). On Windows or Mac, build inside a Vagrant VM.

### A.1 Prerequisites

| Platform | Notes |
|---|---|
| Windows 10 | Recommended. Install [VirtualBox](https://www.virtualbox.org/), then [Vagrant](https://developer.hashicorp.com/vagrant/install). |
| Mac (Intel) | Use Homebrew (below). |
| Mac (M1/M2) | See [this guide](https://www.unixarena.com/2022/09/virtual-machine-on-apple-mac-chip-m1-m2-fusion-vagrant.html/). |
| Mac (M3) | Not supported by Vagrant. |

**Mac (Intel):**

```bash
brew install --cask virtualbox vagrant vagrant-manager
```

### A.2 Set up the build VM (Windows/Mac)

Create and configure the VM:

```bash
mkdir vagrant_machine && cd vagrant_machine

vagrant box add ubuntu/jammy64
vagrant init ubuntu/jammy64

vagrant plugin install vagrant-scp
vagrant plugin install vagrant-disksize
```

Add to the `Vagrantfile`:

```ruby
config.vm.boot_timeout = 6000

config.vm.provider "virtualbox" do |vb|
  vb.memory = "4096"
end
```

Boot and log in:

```bash
vagrant reload --provision
vagrant up
vagrant ssh
```

Copy your definition file into the VM (see `definition_files/` in this repo for an example):

```bash
vagrant scp r450.python310.ubuntu.22.04.def :r450.python310.ubuntu.22.04.def
```

Install build dependencies **inside the VM**:

```bash
sudo apt-get update && sudo apt-get install -y \
  build-essential uuid-dev libgpgme-dev squashfs-tools \
  libseccomp-dev wget pkg-config git cryptsetup-bin
```

Install Go (must be > 1.20 — check with `go version`):

```bash
sudo apt install golang-go

echo 'export GOPATH=${HOME}/go' >> ~/.bashrc
echo 'export PATH=/usr/local/go/bin:${PATH}:${GOPATH}/bin' >> ~/.bashrc
source ~/.bashrc
```

Build and install Singularity:

```bash
git clone https://github.com/hpcng/singularity.git
cd singularity && git checkout v3.8.5

./mconfig && make -C ./builddir && sudo make -C ./builddir install
```

Verify, and optionally enable completion:

```bash
singularity version
singularity exec library://alpine cat /etc/alpine-release

. /usr/local/etc/bash_completion.d/singularity
```

### A.3 Build the container

```bash
sudo singularity build r450.python310.ubuntu.22.04.sif r450.python310.ubuntu.22.04.def
```

Test the image before publishing it — start a shell, check `R --version` and
`python3 --version`, and confirm the packages you added are importable.

### A.4 Publish to DockerHub

Convert the image to a sandbox, tar it, and import it into Docker:

```bash
singularity build --sandbox my_sandbox/ r450.python310.ubuntu.22.04.v3.sif
cd my_sandbox/
tar -cvf r450.python310.ubuntu.22.04.v3.tar .

sudo snap install docker
sudo docker login -u <username>

sudo docker import r450.python310.ubuntu.22.04.v3.tar r450.python310.ubuntu.22.04.v3:latest
sudo docker tag r450.python310.ubuntu.22.04.v3:latest boeings/r450.python310.ubuntu.22.04.v3:latest
sudo docker push boeings/r450.python310.ubuntu.22.04.v3:latest
```

Once pushed, the new image is reachable via `singularity pull` as in
[step 1.2, Option B](#12-get-the-container-image).

---

## Links

- [DockerHub image](https://hub.docker.com/repository/docker/boeings/r450.python310.ubuntu.22.04)
- [Apptainer install docs](https://apptainer.org/admin-docs/master/installation.html)
