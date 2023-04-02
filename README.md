# Configuring a cluster for OpenMPI

<p align="center">
  <img width="300" height="280" src="https://miro.medium.com/v2/resize:fit:300/1*lkzY0KzcU-ra9bqSHa7Kcw.png">
</p>


The following report describes the configurations and detailed procedures for setting up a cluster where applications with MPI can be executed in parallel and distributed. There will be two configuration sides, with the **main node** acting as the server where the execution begins, and the **secondary nodes** running the code and sharing their resources. This way, everyone cooperates to solve a problem.

## Installation of Tools and Libraries

In all nodes the following tools will be required:

```
sudo apt update && sudo apt-get upgrade
sudo apt install build-essential
sudo apt install net-tools
sudo apt install openssh-server
```
* `update` and `upgrade` - First get the latest version from the package list, then download and install the updates.
* `build-essential` - Meta-packages that are essential to compile a program.
* `net-tools` - A collection of programs that form the base set of the NET-3 networking distribution for the Linux operating system.
* `openssh-server` - OpenSSH is a powerful collection of tools for the remote control of, and transfer of data between, networked computers.

### Main node - Network File System (NFS)
NFS allows a system to share directories and files with others over a network. By using NFS, users and programs can access files on remote systems almost as if they were local files.

Some of the most notable benefits that NFS can provide are:

* Local workstations use less disk space because commonly used data can be stored on a single machine and still remain accessible to others over the network.

* There is no need for users to have separate home directories on every network machine. Home directories could be set up on the NFS server and made available throughout the network.

```
sudo apt install nfs-kernel-server
```

* `nfs-kernel-server` On the Main node, install this package, which will allow you to share your directories. 

### Secondary nodes
An NFS file share is mounted on a client machine, making it available just like folders the user created locally. NFS is particularly useful when disk space is limited and you need to exchange public data between client computers.

```
sudo apt install nfs-common
```

* `nfs-common` On the Secondary Nodes, we need to install this package, which provides NFS functionality without including any server components.

## OpenMPI installation

The installation of OpenMPI has to be made on both ,the main and secondary nodes.

### Installation of dependencies
The [OpenMPI Project](https://www.open-mpi.org/) is an open source Message Passing Interface implementation that is developed and maintained by a consortium of academic, research, and industry partners. Open MPI is therefore able to combine the expertise, technologies, and resources from all across the High Performance Computing community in order to build the best MPI library available. Open MPI offers advantages for system and software vendors, application developers and computer science researchers. 

First we install the high-performance message passing library -- binaries, header files and compiler wrappers needed to compile and link programs with libopenmpi, and the development files for the GTK+ library

```
sudo apt-get install openmpi-bin openmpi-common libopenmpi-dev libgtk2.0-dev
```

In some cases the `openmpi-common` command fails, because it is already included in the `openmpi-bin` package so the following commands can be executed, the second one checks that `openmpi-common` is already installed

```
sudo apt-get install openmpi-bin libopenmpi-dev libgtk2.0-dev
sudo apt-get install openmpi-common
```

### OpenMPI source compilation
You must download the latest stable version of OpenMPI with the .tar.bz2 extension.

See: [OpenMPI: Version 4.1](https://www.open-mpi.org/software/)

```
tar -xvf openmpi-4.1.5.tar.bz2
cd openmpi-4.1.5
./configure --prefix="$HOME/.openmpi"
make
sudo make install
```

To export to environment variables and use the binaries from the compiled binary, use the following commands:

```
export PATH="$PATH:$HOME/.openmpi/bin"
export LD_LIBRARY_PATH="LD_LIBRARY_PATH:$HOME/.openmpi/lib"
```

This export will not be permanent and will only work in the current shell session, to make it permanent you must export the file to the shell script that starts when you open the terminal, in my case ZSH.

```
echo export PATH="$PATH:$HOME/.openmpi/bin" >> .zshrc 
echo export LD_LIBRARY_PATH="LD_LIBRARY_PATH:$HOME/.openmpi/lib"  >> .zshrc 
```

## SSH configuration

SSH, or Secure Shell, is a remote administration protocol that allows users to control and modify their remote servers over the Internet through an authentication mechanism.

It provides a mechanism to authenticate a remote user, transfer input from the client to the host and relay the output back to the client.

If you are working with SSH for the first time, you must create a folder that stores the keys, as follows

```
mkdir $HOME/.ssh
chmod 700 $HOME/.ssh
```

* `Note: ` Do this step on the main and secondary nodes.

Ssh-keygen is a tool for creating new authentication key pairs for SSH. Such key pairs are used for automating logins, single sign-on, and for authenticating hosts.

### Main node

To generate an authentication key on the main node, use:

```
ssh-keygen -t rsa
```

Once the command has been executed, the following data is requested:

* `Key Path` - $HOME/.ssh/id_rsa_main, the path to your ssh file and a key name
* `Password` - pass, A password, pass is just an example you can use whatever you want

### Secondary nodes

The following commands can be used to configure two or more secondary nodes

To generate an authentication key on the secondary nodes, use:

```
ssh-keygen -t rsa
```

In the first secondary node

* `Key Path` - $HOME/.ssh/id_rsa_sec1
* `Password` - pass_sec1

In the second secondary node

* `Key Path` - $HOME/.ssh/id_rsa_sec2
* `Password` - pass_sec2

### Setting key permissions

In the main and secondary nodes we create the file containing the keys that are authorised for a communication

```
cd $HOME/.ssh
touch authorized_keys
chmod 664 authorized_keys
```

### Main node

Copy all the `.pub` files, which are the public keys, from the secondary nodes into the `.ssh/` folder of the main node and then add the keys to the `authorized_keys` file. To add use the following:

```
cat $HOME/.ssh/id_rsa_sec1.pub >> $HOME/.ssh/authorized_keys
cat $HOME/.ssh/id_rsa_sec2.pub >> $HOME/.ssh/authorized_keys
```

### Secondary nodes

In the secondary nodes, copy only the `.pub` from the main node and add the key to the `authorized_keys` file.

```
cat $HOME/.ssh/id_rsa_main.pub >> $HOME/.ssh/authorized_keys
```

### sshd server system-wide configuration file

[sshd(8)](https://man.openbsd.org/sshd)  reads configuration data from /etc/ssh/sshd_config (or the file specified with -f on the command line). The file contains keyword-argument pairs, one per line. Lines starting with '#' and empty lines are interpreted as comments. Arguments may optionally be enclosed in double quotes (") in order to represent arguments containing spaces. 

In both we open the sshd_config file and verify that the following parameters are enabled

```
sudo vim /etc/ssh/sshd_config
```

* `PubkeyAuthentication ` yes (It is usually located close to line 37.)
* `RSAAuthentication ` yes (It is usually not found and must be added.)

Reset the service to load the new configurations.

```
sudo service ssh restart
```

### Testing the ssh configuration

The SSH command consists of 2 distinct parts:
```
ssh user@host
```
* `user` represents the account you wish to access. For example, you may want to access the root user, or in our the user of the main node or secondary nodes

* `host` refers to the computer you want to access. This can be an IP address (e.g., 0.0.0.1), to get the IP's you can use the command `ifconfig`.

Main Node (KDE Neon and Shell Zsh)  |  Secondary Node (Ubuntu and Shell Bash)  
:----------------------------------:|:----------------------------:
![](https://i.ibb.co/94GFNyL/Main.png)  |  ![](https://i.ibb.co/5cd2syG/Sec1.png)

## NFS configuration

### Main node
Create a folder in which files can be shared or used to run OpenMPI source code. In our case we will create this folder on the desktop
```
sudo mkdir -p $HOME/Desktop/sharedMPI
sudo chown nobody:nogroup $HOME/Desktop/sharedMPI
sudo chmod 777 $HOME/Desktop/sharedMPI
```

The `/etc/exports` file controls which file systems are exported to remote hosts and specifies options. 

A line for an exported file system has the following structure: 

```
<export> <host1>(<options>) <hostN>(<options>)...
```

To edit we make use of the following command:

```
sudo vim /etc/exports
```

The secondary nodes that are allowed to access files on the main node need to be entered into `/etc/exports`. And we add the following lines, changing `sec1_ip` and `sec2_ip` to their respective IP's

```
$HOME/Desktop/sharedMPI sec1_ip(rw,sync,no_subtree_check)
$HOME/Desktop/sharedMPI sec2_ip(rw,sync,no_subtree_check)
```

After modifying `/etc/exports`, the file needs to be re-read with exportfs, `-a` for all (of `/etc/exports`). 

```
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

* `restart` option is a shorthand way of stopping and then starting NFS. This is the most efficient way to make configuration changes take effect after editing the configuration file for NFS. 

## Firewall status
A firewall is what controls what - and most importantly what - is not allowed to pass through those ports, we need to disable it to allow NFC to share a directory on our LAN, check with:

```
sudo ufw status
```

To disable:
```
sudo ufw disable
```

Then can be enable it again:
```
sudo ufw enable
```
### Secondary nodes

In the secondary nodes just create the file and mount, change `main_ip` to the IP of the main node.

```
sudo mkdir -p $HOME/Desktop/sharedMPI
sudo mount main_ip:$HOME/Desktop/sharedMPI $HOME/Desktop/sharedMPI
```

### Example of a result

[](https://i.ibb.co/tCnBdBV/1-test.gif) 



### Host configurations
This file is used to obtain a relation between a hostname and an IP address: in each line of `/etc/hosts`, an IP address and its corresponding hostnames are specified, so that a user does not have to remember addresses but hostnames.

### Main node

```
sudo vim /etc/hosts
```

Add the following lines and comment the rest with a `#`:

```
sec1_ip sec_1
sec1_ip sec_2
```

### Secondary nodes
```
sudo vim /etc/hosts
```

Add the following lines and comment the rest with a `#`:

```
main_ip main
```

## Final results

Compile a OpenMPI program with:
```
mpicc file.c -o ./outputfile
mpirun --hostfile etc/hosts -np 6 outputfile
```

[](https://i.ibb.co/tCnBdBV/1-test.gif) 

---
### Contributors
- Chillitupa Quispihuanca, Alfred Addison
- Curo Quispe, Iris Rocio
- Muñoz Curi, Rayver Aimar
- Quispe Salcedo, Josep Marko
