# Singularity image to run R and python
This repository provides a container solution to run R and python together for data analyses. 

# Downloading and Using the Singularity Image from Dockerhub and setting up the file structure

Create the following folder structure on your linux system:

```
-demo
    ├── scripts
    ├── envs
    ├── workdir
    └── data
```
          
This singularity container is available on dockerhub: [Link](https://hub.docker.com/repository/docker/boeings/r450.python310.ubuntu.22.04). Within the institute, it also is available at the path in the file systme below. There is no need to copy the container.
```
/flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif
```

The image can be imported as follows ( `--cleanenv` is optional in many settings):

# Start the container on an external file sytem

```{bash}
## For R 4.5.0 (downloading the docker image to your folder - see below the option to access the singularity environment on the institute's system)

## Go to envs folder
cd envs

ml Singularity/3.6.4
singularity pull docker://boeings/r450.python310.ubuntu.22.04

cd ../workdir
```

## Starting the singularity container
```{bash}
singularity shell --bind <local_data_directory_1>:/local data directory_1,<local_data_directory_2>:/local data directory_2 ../envs/r450.python310.ubuntu.22.04_latest.sif
```

If the above command causes issues, add the `--cleanenv` flag


```{bash}
# Or create venv environment in the current folder (change path if you'd like to store the venv environment files elsewhere)
python3.10 -m venv ../envs/demo_venv_310
```

## Activate venv environment
```{bash}
source ../envs/demo_venv_310/bin/activate
```

You can now populate the venv environment yourself using `pip install [python package name]` or download the provided venv.lock file and do
```pip install -r venv.lock```

## Start python
```{bash}
python
```


# Starting the Singularity Image on the Institute's File System

Set up Ubuntu singularity Instance on a Mac. This can be done in a similary fashion on a LINUX or Windows system. 

The ready-build singularity instance can be started on nemo like so:

## Starting the singularity container

```{bash}
cd workdir

ml Singularity/3.6.4

## --cleanenv is optional. It prevents the transfer of host system environmental variables into the singularity container.

## For R 4.5.0
singularity shell --bind /nemo:/nemo,/camp:/camp,/flask:/flask /flask/apps/containers/all-singularity-images/r450.python310.ubuntu.22.04.v3.sif

In case there are issues with the above commands in your area, try adding the `--cleanenv` flag to the `singularity shell` command.
```

## Activating the venv environment

Either create a new venv environment in the envs folder:
```{bash}
python3.10 -m venv ../envs/demo_venv_310
```

Or simply activate a venv environment if it already exists in the envs folder or elsewhere on your file system:

```{bash}
source ../envs/demo_venv_310/bin/activate
```

You can now populate the venv environment yourself using `pip install [python package name]` or download the provided venv.lock file and do
```pip install venv.lock```

## Setting R caches
```{R}
export RENV_PATHS_LIBRARY=/nemo/stp/babs/working/boeings/package_caches/R/library/
export RENV_PATHS_CACHE=/nemo/stp/babs/working/boeings/package_caches/renv/cache/
export RENV_PATHS_ROOT=/nemo/stp/babs/working/boeings/package_caches/renv/
```

## Starting the jupyter notebook
Now we're ready to start the jupyter notebook server

```{bash}
jupyter notebook --no-browser --port=8888 --ip=127.0.0.1
```

Copy the URL that's displayed in the terminal window and paste it into a webbroswer on your computer. Here is an <b>example</b> what that URL can look like:
`http://127.0.0.1:8888/tree?token=6a671b92eac9d09f28971964ba1751147007844d2a688817`

In your webbrowser, after copying and pasting the URL displayed on the cluster, open the 
`/singularityInstance/example_python_R_notebooks/Seurat_to_anndata_conversion.ipynb` file by double-clicking it. To avoid clashes with other users, save the notebook under a new name.

Once you're down with work
* Save your jupyter notebook
* Close the webbrowser window
* Go to the terminal and shut-down the jupyter server using `Ctrl-C`

## Trouble shooting
If the jupyterserver does not start, it might be necessary to install on the cluster, in the venv environment the following packages.

```{}
pip install jupyter notebook jupyterlab
```

If you see something like below, you are likely to have a browser window still open that tries to connect to the jupyter server you've just shut down. 
```
channel 3: open failed: connect failed: Connection refused
channel 3: open failed: connect failed: Connection refused
channel 3: open failed: connect failed: Connection refused
channel 3: open failed: connect failed: Connection refused
channel 3: open failed: connect failed: Connection refused
```

When starting up a new project, we might need to install the jupyter notebook kernel in the renv package environment like so (after starting R in the singularity container).
```{r}
renv::install('IRkernel')
renv::snapshot()
```
This only needs to be done once. 




# Instructions to create a new, modified container

## Vagrant installation 

Please note, if you are using a **Mac with an M1/M2 chip**, you'll have to modify the vagrant installation procedure as described [here](https://www.unixarena.com/2022/09/virtual-machine-on-apple-mac-chip-m1-m2-fusion-vagrant.html/). At present, vagrant does not work with a Mac M3 chip. 

The easiest at present is to use a Windows 10 machine as outlined [here](https://medium.com/@botdotcom/installing-virtualbox-and-vagrant-on-windows-10-2e5cbc6bd6ad). 

If vagrant is not installed on your Windows PC, you need to start with this. 
Before installing vagrant, you need to install Oracle Virtual Box on your PC. 

Download Vagrant and install: [Link](https://developer.hashicorp.com/vagrant/install)

Start the vagrant virtual machine

Open the Windows command prompt do

```{bash}
# Navigate to a suitable directory

mkdir vagrant_machine
cd vagrant_machine

vagrant destroy

vagrant box add ubuntu/jammy64
vagrant init ubuntu/jammy64

# Install vagrant scp
vagrant plugin install vagrant-scp
vagrant plugin install vagrant-disksize

# Edit vagrant file
Add to the vagrant file:

```{bash}
config.vm.boot_timeout = 6000

config.vm.provider "virtualbox" do |vb|
  vb.memory = "4096" # Increase memory
end
```
Once the vagrant file is edited, run 

```{bash}
vagrant reload --provision
```

# Starting the vagrant LINUX machine
```{bash}
vagrant up
```


# Upload def file into instance via vagrant scp (from the host sytem - exit vagrant)

```{bash}
vagrant scp r450.python310.ubuntu.22.04.def :r450.python310.ubuntu.22.04.def
```


# Log into the machine
vagrant ssh

```

```{bash}
Alternative (Mac non M1/M2/M3)
brew install --cask virtualbox && \
  brew install --cask vagrant && \
  brew install --cask vagrant-manager
```

Make sure no old virtual machine is present

```{bash}
cd vm-singularity

vagrant destroy && \
    rm Vagrantfile
```

Install Ubuntu, start virtual machine and ssh into it

```{bash}
## Get ubuntu version 23
# export VM=ubuntu/mantic64 && \
#    vagrant init $VM && \
#    vagrant up && \
#    vagrant ssh

# Correct the Vagrant box name
export VM=ubuntu/jammy64

# Initialize the Vagrant environment
vagrant init $VM

# Start the Vagrant virtual machine
vagrant up


# SSH into the Vagrant virtual machine
vagrant ssh



```

## Install latest singularity version in the vagrant machine
Instructions https://apptainer.org/admin-docs/master/installation.html

### Install dependencies in the VM
```{bash}
sudo apt-get update && sudo apt-get install -y \
    build-essential \
    uuid-dev \
    libgpgme-dev \
    squashfs-tools \
    libseccomp-dev \
    wget \
    pkg-config \
    git \
    cryptsetup-bin
```

Install go

```{bash}
sudo apt update
sudo apt upgrade
sudo apt search golang-go
sudo apt search gccgo-go

sudo apt install golang-go
```

go version #needs to be > 1.20
Export go path

```{bash}
echo 'export GOPATH=${HOME}/go' >> ~/.bashrc && \
    echo 'export PATH=/usr/local/go/bin:${PATH}:${GOPATH}/bin' >> ~/.bashrc && \
    source ~/.bashrc
```

## Install Singularity

```{bash}
git clone https://github.com/hpcng/singularity.git && \
    cd singularity && \
    git checkout v3.8.5
```

## compile singularity
```{bash}
./mconfig && \
    make -C ./builddir && \
    sudo make -C ./builddir install

Check version
```

```{bash}
singularity version
```

add bash completion
```{bash}
. /usr/local/etc/bash_completion.d/singularity
```

### Test
```{bash}
singularity exec library://alpine cat /etc/alpine-release
```

## Create definition file

```{bash}

sudo singularity build r431.ubuntu.22.04.sif r431.ubuntu.22.04.def

See example def file in the definition_files folder
```

## updating docker in vagrant
```{bash}
sudo apt-get install docker.io 
```

## Export from vagrant VM

## Export environment
```{bash}
vagrant ssh-config > config.txt
vagrant scp :/home/vagrant/r442.ubuntu.22.04.* .

```


## Start Singularity Image
```{bash}
# For some R-installations, such as Seurat::SCTransform the following sequence is required:
# mkdir -p ~/.R
# echo 'CXX17 = g++-11 -std=gnu++17 -fPIC' > ~/.R/Makevars

ml Singularity/3.11.3
singularity shell --bind  /nemo:/nemo,/camp:/camp /nemo/stp/babs/working/boeings/singularity_images/r431.ubuntu.22.04.sif;
```

## Create dockerhub image
In the vagrant virtual machine do the following

```{bash}
singularity build --sandbox my_sandbox/ r450.python310.ubuntu.22.04.v3.sif

cd my_sandbox/
tar -cvf r450.python310.ubuntu.22.04.V3.tar .

sudo snap install docker
sudo docker login -u [username]

sudo docker import r450.python310.ubuntu.22.04.v3.tar r450.python310.ubuntu.22.04.v3:latest
sudo docker tag r450.python310.ubuntu.22.04.v3:latest boeings/r450.python310.ubuntu.22.04.v3:latest

# List your images to confirm the import
sudo docker images

# Tag the imported image with your username
sudo docker tag r450.python310.ubuntu.22.04.v3:latest boeings/r450.python310.ubuntu.22.04.v3:latest

# Now push it
sudo docker push boeings/r450.python310.ubuntu.22.04.v3:latest

```
