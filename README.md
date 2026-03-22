# R + Python Singularity Container

Run R and Python together for data analysis using a pre-built Singularity image.

## Contents

- [Quick Start](#quick-start)
- [Starting the Container](#starting-the-container)
- [Python venv Setup](#python-venv-setup)
- [Institute Cluster (NEMO) Setup](#institute-cluster-nemo-setup)
- [Troubleshooting](#troubleshooting)
- [Building a New Container](#building-a-new-container)
  - [Prerequisites](#prerequisites)
  - [Windows: Vagrant Setup](#windows-vagrant-setup)
  - [Install Singularity in the VM](#install-singularity-in-the-vm)
  - [Build the Container](#build-the-container)
  - [Push to DockerHub](#push-to-dockerhub)
- [Links](#links)

---

## Quick Start

### 1. Set up folder structure

```
demo/
├── scripts/
├── envs/
├── workdir/
└── data/
```

### 2. Get the container

**Option A — Download from DockerHub:**

```bash
cd envs
ml Singularity/3.6.4
singularity pull docker://boeings/r450.python310.ubuntu.22.04
```

**Option B — Use the institute path (no download needed):**

```
/flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif
```

### 3. Clone the scripts

```bash
cd ../scripts
git clone git@github.com:decusInLabore/singularityInstance.git
```

---

## Starting the Container

```bash
singularity shell --bind <local_dir_1>:/local_dir_1,<local_dir_2>:/local_dir_2 \
  ../envs/r450.python310.ubuntu.22.04_latest.sif
```

> Add `--cleanenv` if you encounter environment variable conflicts.

---

## Python venv Setup

**Create a new venv:**

```bash
python3.10 -m venv ../envs/demo_venv_310
```

**Activate it:**

```bash
source ../envs/demo_venv_310/bin/activate
```

**Install packages:**

```bash
pip install <package>
# or restore from lock file:
pip install -r venv.lock
```

---

## Institute Cluster (NEMO) Setup

### 1. SSH with port forwarding

```bash
ssh -L 8888:localhost:8888 boeings@babs003.nemo.thecrick.org
```

### 2. Free port 8888 if needed

```bash
lsof -ti:8888 | xargs kill -9
```

### 3. Start the container

```bash
cd scripts
ml Singularity/3.6.4
singularity shell --bind /nemo:/nemo,/camp:/camp,/flask:/flask \
  /flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif
```

> Add `--cleanenv` if you encounter issues.

### 4. Activate venv

```bash
source ../envs/demo_venv_310/bin/activate
```

### 5. Set R caches

```bash
export RENV_PATHS_LIBRARY=/nemo/stp/babs/working/boeings/package_caches/R/library/
export RENV_PATHS_CACHE=/nemo/stp/babs/working/boeings/package_caches/renv/cache/
export RENV_PATHS_ROOT=/nemo/stp/babs/working/boeings/package_caches/renv/
```

### 6. Start Jupyter

```bash
jupyter notebook --no-browser --port=8888 --ip=127.0.0.1
```

Copy the URL from the terminal and paste it into your browser. Example:

```
http://127.0.0.1:8888/tree?token=6a671b92eac9d09f28971964ba1751147007844d2a688817
```

Open `singularityInstance/example_python_R_notebooks/Seurat_to_anndata_conversion.ipynb`. Save it under a new name before editing.

**When done:**
1. Save your notebook.
2. Close the browser.
3. Shut down the server with `Ctrl-C`.

---

## Troubleshooting

**Jupyter won't start:**

```bash
pip install jupyter notebook jupyterlab
```

**Repeated `Connection refused` messages in terminal:**
Close any browser tabs still connected to a previous Jupyter session.

**Install R kernel (once per new project):**

```r
renv::install('IRkernel')
renv::snapshot()
```

---

## Building a New Container

### Prerequisites

| Platform | Notes |
|---|---|
| Windows 10 | Recommended. Install [VirtualBox](https://www.virtualbox.org/) then [Vagrant](https://developer.hashicorp.com/vagrant/install). |
| Mac (Intel) | Use Homebrew (see below). |
| Mac (M1/M2) | See [this guide](https://www.unixarena.com/2022/09/virtual-machine-on-apple-mac-chip-m1-m2-fusion-vagrant.html/). |
| Mac (M3) | Not supported by Vagrant. |

**Mac (Intel) install:**

```bash
brew install --cask virtualbox vagrant vagrant-manager
```

---

### Windows: Vagrant Setup

```bash
mkdir vagrant_machine && cd vagrant_machine

vagrant box add ubuntu/jammy64
vagrant init ubuntu/jammy64

vagrant plugin install vagrant-scp
vagrant plugin install vagrant-disksize
```

Add to `Vagrantfile`:

```ruby
config.vm.boot_timeout = 6000

config.vm.provider "virtualbox" do |vb|
  vb.memory = "4096"
end
```

Then:

```bash
vagrant reload --provision
vagrant up
vagrant ssh
```

**Upload the `.def` file into the VM:**

```bash
vagrant scp r450.python310.ubuntu.22.04.def :r450.python310.ubuntu.22.04.def
```

---

### Install Singularity in the VM

**Install dependencies:**

```bash
sudo apt-get update && sudo apt-get install -y \
  build-essential uuid-dev libgpgme-dev squashfs-tools \
  libseccomp-dev wget pkg-config git cryptsetup-bin
```

**Install Go:**

```bash
sudo apt install golang-go

echo 'export GOPATH=${HOME}/go' >> ~/.bashrc
echo 'export PATH=/usr/local/go/bin:${PATH}:${GOPATH}/bin' >> ~/.bashrc
source ~/.bashrc
```

> Go version must be > 1.20. Check with `go version`.

**Install Singularity:**

```bash
git clone https://github.com/hpcng/singularity.git
cd singularity && git checkout v3.8.5

./mconfig && make -C ./builddir && sudo make -C ./builddir install
```

**Verify:**

```bash
singularity version
singularity exec library://alpine cat /etc/alpine-release
```

**Enable bash completion:**

```bash
. /usr/local/etc/bash_completion.d/singularity
```

---

### Build the Container

```bash
sudo singularity build r450.python310.ubuntu.22.04.sif r450.python310.ubuntu.22.04.def
```

See the `definition_files/` folder for an example `.def` file.

---

### Push to DockerHub

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

---

## Links

- [DockerHub image](https://hub.docker.com/repository/docker/boeings/r450.python310.ubuntu.22.04)
- [Apptainer install docs](https://apptainer.org/admin-docs/master/installation.html)
