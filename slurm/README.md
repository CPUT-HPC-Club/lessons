# Slurm

1. [Prequisites](#prequisites)

    1. [NTP](#ntp)
    2. [Munge](#munge)

2. [Slurm](#slurm-1)

    1. [Controller](#headnode-controller)
    2. [Compute](#compute-nodes-can-include-headnode)
    3. [Usage](#usage)

        i.   [Script](#sample-script)

        ii.  [Interactive](#interactive)

        iii. [Commands](#commands)

3. [Errors](#additional-errors-and-bugs)

Slurm is a “work load manager”

Slurm is used to provide a queuing system and scheduler for supercomputers.

## Prequisites

1. Munge (installed and configured)
2. NTP (required for munge. Network Time Protocol)
3. pdsh (optional to check date)

### NTP

The munged daemon will not be able to authenticate Slurm-related communication unless all nodes have the same time configured.

```bash
# install NTP or chrony
# Steps for NTP

dnf install epel-release

# Set server
vim /etc/ntp.conf

# Enable and start service
systemctl enable ntpd
systemctl start ntpd

# check list of peers
ntpq -p

# Set date
ntpdate <NTP server name or IP>
```

## Munge

Munge is used by Slurm and **others** to provide an authentication service for the slurmd daemon that runs on the compute nodes.

1. Install everywhere

```bash
# Create a munge user everywhere
sudo dnf install munge -y
```

2. CHECK

The munge user must have the same UID everywhere in the system.

```bash
# First, check the UID and GID of the munge user on the login node, and make a note of the two numbers.

getent passwd munge

# Then check on the compute nodes

pdsh -w com[1-2] getent passwd munge

# The first number is the userID (UID), the second is the groupID (GID).

# Use the UID that you recorded on the login node, and change the compute nodes to match.

pdsh -w com[1-2] sudo usermod --uid 979 munge
pdsh -w com[1-2] getent passwd munge

# Now the group
pdsh -w com[1-2] sudo groupmod --gid 979 munge
pdsh -w com[1-2] getent passwd munge

# Verify
pdsh -w head,com[1-2] getent passwd munge
```

3. Generate a key on headnode

```bash
# enter root
sudo su

# This should be generated on your headnode and copied to the other nodes.

/usr/sbin/create-munge-key
chown munge: /etc/munge/munge.key
chmod 0400 /etc/munge/munge.key

# The munged daemon is very picky about permissions and ownership of thekey
```

4. Copy the key

For the code below to work, *passwordless* ssh for **root** has to have been setup.

```bash
# for root key
sudo su
ssh-keygen -t ed25519 -C "your.email@example.com"
cd ~
# copy your private key and paste it on the other nodes
cat .ssh/id_ed25519

# copy your public key and paste it on the other nodes
cat .ssh/id_ed25519.pub

# make sure to add the key yo authorized_keys
cat id_ed25519.pub >> authorized_keys

# Set permissions
chmod 700 ~/.ssh/
chmod 600 ~/.ssh/authorized_keys
sudo restorecon -R -v ~/.ssh
```

Run this script to copy over munge key

```bash
# This loop should work
for i in $(seq 1 2); do
  scp /etc/munge/munge.key com${i}:/etc/munge \
    && ssh com${i} "sudo chown -R munge:munge /etc/munge"
done

# exit root
exit
```

5. Start munge

```bash
sudo systemctl enable munge
sudo systemctl start munge
```

On start-up [error](#additional)

## Slurm

### Headnode (Controller)

Needs

• slurm

• slurm-slurctld

• slurm-slurmdbd(optional)

• munge

```
sudo dnf install slurm -y
sudo dnf install slurm-slurmctld -y
```

Slurm will have installed on your system with a default configuration file, but you will need to change this.

[Slurm_Configuring_Tool](https://slurm.schedmd.com/configurator.html) can help with is

Make sure that the files match across thec luster, or use “configless’ Slurm, where each slurmd gets the configuration from slurmctld.

```bash
# Get settings from Slurm_Configuring_Tool
sudo vim /etc/slurm/slurm.conf

# Start Controller
systemctl enable slurmctld
systemctl start slurmctld

```

### Compute nodes (Can include headnode)

Needs

• slurm

• slurm-slurmd

• munge

```
sudo dnf install slurm -y
sudo dnf install slurm-slurmd -y
```

For the headnode (controller) copy over the conf file

```bash
# From headnode copy over
for i in $(seq 1 2); do
  sudo scp /etc/slurm/slurm.conf com${i}:/etc/slurm \
    && ssh com${i} "sudo chown cput:cput /etc/slurm/slurm.conf && sudo chmod 644 /etc/slurm/slurm.conf"
done

sudo systemctl enable slurmd
sudo systemctl start slurmd
```

## Usage

There is 3 ways to use slurm

1. Batch script     -   Submit job to queue

2. Interactive      -   Take interactive control of resources

3. srun             -   Direct run

### Sample script

Batch script Example

```bash
#!/bin/sh
#This file is called submit-script.sh
#SBATCH --partition=shared       # default "shared", if not specified
#SBATCH --time=0-04:30:00       # run time in days-hh:mm:ss
#SBATCH --nodes=1               # require 1 nodes
#SBATCH --ntasks-per-node=64    # cpus per node (by default, "ntasks"="cpus")
#SBATCH --mem=4000             # RAM per node in megabytes
#SBATCH --error=job.%J.err
#SBATCH --output=job.%J.out
# Make sure to change the above two lines to reflect your appropriate
# file locations for standard error and output

# Now list your executable command (or a string of them).
# Example for code compiled with a software module:
module load mpimodule
srun --mpi=pmix -n 64 /home/username/mpiprogram

# Use mpirun in slurm
# mpirun -n 64 ./home/cput/hpl/xhpl
```

Use `sbatch` to launch script

```bash
sbatch myScript.sh
```

### Interactive

Use an interactive session to reserve resouces

```bash
# Check on cluster status
sinfo

# check the queue
squeue

# Request resources
salloc

# Common options
# --cpus-per-task=<count>
# --error=<filename>
# --exclusive
# --export=<values>
# --job-name=<name>
# --mem=<MB>
# --mem-per-cpu=<MB>
# -N <node count>
# -n <task count>
# --nodelist=<names>
# --output=<name>
# --partition=<name>
# --time=<HH:MM:SS>

# E.g.
salloc --partition=debug --time=10:00 -N 2 -n 64 --job-name=cput
```

### Commands

```bash
# Check on cluster
sinfo

# Submit script
sbatch submit-script.sh

# Check status
squeue -u username
squeue -u cput

# Kill job
scancel job#

# Pause / resume Job
scontrol hold job#
scontrol release job#

# Request resources
srun

# E.g.
srun --partition=debug --time=10:00 -N 2 -n 64 --job-name=cput

# Common options
# --cpus-per-task=<count>
# --error=<filename>
# --exclusive
# --export=<values>
# --job-name=<name>
# --mem=<MB>
# --mem-per-cpu=<MB>
# -N <node count>
# -n <task count>
# --nodelist=<names>
# --output=<name>
# --partition=<name>
# --time=<HH:MM:SS>
```

Additional

```bash
#SBATCH --mem-per-cpu=4G            //set mem per cpu
#SBATCH --mem=32G                   //total mem per node

#Test limit (replace 4 with max available threads/cores)
srun -n4 -l /bin/hostname
```

Additional guides

[slurm](https://slurm.schedmd.com/quickstart.html) guide


## Additional errors and bugs

On munge error

```bash
# If you get a error on the start-up about permissions use this one to capy key and permissions

for i in $(seq 1 2); do
  scp /etc/munge/munge.key com${i}:/etc/munge \
    && ssh com${i} "sudo chown -R munge:munge /etc/munge /var/lib/munge /var/log/munge && sudo chmod 400 /etc/munge/munge.key && sudo chmod 700 /etc/munge"
done

# Check status
systemctl status munge

# Restart
sudo systemctl restart munge
```

On slurm error

```bash
# Check controller status
systemctl status slurmctld

# Restart controller
sudo systemctl restart slurmctld

# Check node status
systemctl status slurmd

# Restart node
sudo systemctl restart slurmd

# Restore nodes if drained
sudo scontrol update NodeName=head State=RESUME
sudo scontrol update NodeName=com[1-2] State=RESUME
```
#### Management

```
# The Slurm controller daemon manages the queue
slurmctld

# The Slurm daemon launches job tasks
slurmd

# The Slurm database daemon allows you to keep accounting data.
slurmdbd
```

#### Testing

```bash
# pdsh -options (w for nodes) nodes <command> 
pdsh -w head,com[1-2] date
```