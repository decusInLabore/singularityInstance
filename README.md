# Downloading and Using the Singularity Image from Dockerhub
This singularity container is available on dockerhub: [Link](https://hub.docker.com/r/boeings/r450.ubuntu.22.04.docker.image)

The image can be imported as follows:

```{bash}
## For R 4.5.0 (downloading the docker image to your folder - see below the option to access the singularity environment on the institute's system)

ml Singularity/3.6.4
singularity pull docker://boeings/r450.ubuntu.22.04.docker.image

singularity shell --cleanenv --bind /nemo:/nemo,/camp:/camp r450.ubuntu.22.04.docker.image_latest.sif

```

# Starting the Singularity Image with Access to the Institute's File System

Set up Ubuntu singularity Instance on a Mac. This can be done in a similary fashion on a LINUX or Windows system. 

The ready-build singularity instance can be started on nemo like so:

```{bash}
ml Singularity/3.6.4

## --cleanenv is optional. It prevents the transfer of host system environmental variables into the singularity container.

## For R 4.3.1
singularity shell --cleanenv --bind /nemo:/nemo,/camp:/camp /flask/apps/containers/all-singularity-images/r431.ubuntu.22.04.sif;

## For R 4.5.0
singularity shell --cleanenv --bind /nemo:/nemo,/camp:/camp /flask/apps/containers/all-singularity-images/r450.ubuntu.22.04.sif;
```

## Vagrant installation 

Please note, if you are using a **Mac with an M1/M2 chip**, you'll have to modify the vagrant installation procedure as described [here](https://www.unixarena.com/2022/09/virtual-machine-on-apple-mac-chip-m1-m2-fusion-vagrant.html/). At present, vagrant does not work with a Mac M3 chip. 

The easiest at present is to use a Windows 10 machine as outlined [here](https://medium.com/@botdotcom/installing-virtualbox-and-vagrant-on-windows-10-2e5cbc6bd6ad). 

If vagrant is not installed on your Windows PC, you need to start with this. 
Before installing vagrant, you need to install Oracle Virtual Box on your PC. 

Download Vagrant and install: [Link](https://developer.hashicorp.com/vagrant/install)


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
vagrant up
exit

# Upload def file into instance via vagrant scp (from the host sytem - exit vagrant)
vagrant scp r442.ubuntu.22.04.def :r442.ubuntu.22.04.def



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


##Start Singularity Image
```{bash}
# For some R-installations, such as Seurat::SCTransform the following sequence is required:
# mkdir -p ~/.R
# echo 'CXX17 = g++-11 -std=gnu++17 -fPIC' > ~/.R/Makevars

ml Singularity/3.11.3
singularity shell --bind  /nemo:/nemo,/camp:/camp /nemo/stp/babs/working/boeings/singularity_images/r431.ubuntu.22.04.sif;
```
