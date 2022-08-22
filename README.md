# Introduction

## Create & SSH into machine

- first step is downloading virtualbox and vagrant
- second step is downloading the linux box for vagrant ` vagrant init debian/bullseye64 `
- there are preconfigured boxes already available
- this creates a vagrantfile with some content
  - uncomment the public network line so that vm acts as another device on same network as host
- to start the machine with basic settings, just run `vagrant up`
- sometimes the boot time takes too long and a timeout occurs but you can start Virtualbox and check whether that vm is running or not
- once running, run `vagrant ssh` to connect to it via ssh (sometimes this hangs)
  - in this case, halt the machine with `vagrant halt`
  - Open control panel > Programs > turn windows features on or off
  - Disable VirtualMachinePlatorm & Windoes Hyper-V and restart the system
  - This can have an effect on running Docker Desktop
- at this point your machine is up and you are connected to it at /home/vagrant directory
- you can run `logout` on the machine terminal to jump out of connection

## Pinging host and vm

- run `ipconfig` on windows host and look at the ipv4 address in the virtualbox host-only network to find the ip which can be pinged from the vm
- run `ip a` on the debian vm (may have different command on other linux variants to check ip) and look at the first ip after inet of last entry to get the vm ip
- running ping command from host with ip of vm will successfully ping and vice versa

## Install required software

- `sudo apt-get update && sudo apt-get upgrade` to get latest Debian packages
- `sudo apt-get autoremove` to clean up and remove unrequired packages
- `sudo apt-get install build-essential` to install gcc, make etc required to compile python
- `sudo apt-get install zlib1g zlib1g-dev` to install zlib for decompressing data
- `sudo apt install libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libsqlite3-dev` to install additional packages required by python & pip
- `sudo apt-get install libffi-dev` for _ctypes module that is required for ansible
- `sudo apt install vim` for vim so as to be able to edit files easily (vi doesn't allow backspace by default)

## Install Python for Ansible

- `wget https://www.python.org/ftp/python/3.8.12/Python-3.8.12.tgz` to download the python zipped package (find the appropriate version, this one is 3.8.12)
- `tar -xf Python-3.8.12.tgz` to unzip it
- `sudo mv Python-3.8.12 /opt/Python-3.8.12` to move the installation to opt directory and then go inside `/opt/Python-3.8.12` directory
- `./configure --enable-optimizations --enable-shared` to verify dependencies and configure the environment
- the above step may specify errors and if successful, creates a makefile
- `make` to compile makefile
- `sudo make altinstall` to install Python binaries and we use altinstall to not overwrite the default Python3 binaries (this requires sudo even if just "make" does not)
  - above command returned exit code 1 => not sure how to fix
- `sudo ldconfig /opt/Python-3.8.12` to configure dynamic linker run-time bindings
- `python3.8 -V` (capital V) to print version and you should see Python 3.8.12

## Install Pip

- `wget https://bootstrap.pypa.io/get-pip.py` to download pip
- `python3.8 get-pip.py` to install pip (may warn about pip not being on path, in which case follow next sub-point)
  - `export PATH=$HOME/.local/bin:$PATH` will add the pip install dir to PATH where $HOME will be /home/vagrant and $PATH is the rest of the paths configured (there should be no spaces around the '=')
- `python3.8 -m pip install --upgrade pip` to upgrade pip version
- `pip3.8 --version` to check pip version and that its installed properly

## Install Ansible

- `pip3 install --user ansible` to install ansible
  - gave error that "python setup.py egg_info did not run successfully"
  - `pip3 list` to confirm setuptools is installed and otherwise run `pip3 install --user setuptools`
- `ansible --version` to check ansible-core version and if its installed successfully
- `pip3 list` to check version of ansible and ansible-core
- `sudo mkdir ansible` in /etc and create ansible.cfg file by `ansible-config init --disabled > ansible.cfg`

## Create vms as infrastructure

- Ansible manages infrastructure as servers in an inventory file
- We need these servers to exist so we create a vagrantfile in an `inventory` folder that creates 2 app servers
- We set the network type to public network so that they work as new devices on the network but we don't assign them an IP as doing to doesn't make them accessible from host
- Instead we can run `ip a` on the vm to find its ip

## Generating ssh keys
- Navigate to `cd ~/.ssh` of each of ansible host
- Use `sudo ssh-keygen -t rsa` to generate new ssh key
- It asks path of file to save in so name it as `ansible_id_rsa`
- Give it a pass phrase for security like `ansible`
- It will then save that private key in `~/.ssh/ansible_id_rsa` and public key at `~/.ssh/ansible_id_rsa.pub`
- Currently the owner is root but change the ownership of both to `vagrant` user by `sudo chown vagrant:vagrant ansible_id_rsa` and same for the public key
- Now, we need to decide what user of app1 and app2 should be accessible via SSH, and we will choose both `vagrant` and `root`
- Copy the public key and manually paste them in `~/.ssh/authorized_keys` file of app1 and app2 while being `vagrant` user and `root` user separately
- You can change between users with `sudo su - <username>` and while in root, may have to create the .ssh directory and authorized_keys file in it
- After that, doing `sudo ssh -i ansible_id_rsa <username>@<ip of server>` from `~/.ssh` will allow connecting to those machines via ssh after verifying ssh passphrase

## Troubleshooting SSH
- Generate keys in `~/.ssh/`
- In `/etc/ssh`, do `sudo vi sshd_config` and make sure you have the following:
  - PubKeyAuthentication yes
  - RSAAuthentication yes
  - PermitRootLogin yes
  - PasswordAuthentication no
- After updating these, make sure to restart ssh service with `sudo systemctl restart ssh.service`
- Every user has its own home directory with `~/.ssh.authorized_keys` so SSH is also specific to that user

## Ansible config & inventory
- The config file by default exists in /etc/ansible as ansible.cfg
- If it doesn't exist, we can create it as covered in the installation section of the readme
- In ansible.cfg on server, add the `[ssh-connection]` section of the ansible.cfg file here including `host_key_checking`, `pipelining` and `ssh_args`
- The inventory file by default, gets picked up from /etc/ansible/hosts where hosts is the file name
- We can specify an inventory file called hosts as shown here in project
- Running `ansible app -m ping -u vagrant` should be able to ping the hosts
  - app is the group specified in the hosts inventory file
  - -m implies use the ping module
  - -u implies use the vagrant user
  - fails for all but first machine for the first time showing python interpreter issue but works every time after that
  - specifying python interpreter anywhere makes it fail every time
- We can create our own inventory.yml file to override etc/ansible/hosts
- Do `mkdir ansible-learning` in home(~) directory and create inventory.yml file as shown here in project
- Running `ansible app -i ~/ansible-learning/inventory.yml -m ping` will now do something similar to before
  - works exactly like when we run using hosts (including the one-time fail etc)
  - -i specifies the custom inventory file to use, defaults to /etc/ansible/hosts
- 

---
