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
- Skip giving it a pass phrase
- It will then save that private key in `~/.ssh/ansible_id_rsa` and public key at `~/.ssh/ansible_id_rsa.pub`
- Currently the owner is root but change the ownership of both to `vagrant` user by `sudo chown vagrant:vagrant ansible_id_rsa` and same for the public key
- Now, we need to decide what user of app1 and app2 should be accessible via SSH, and we will choose both `vagrant` and `root`
- Copy the public key and manually paste them in `~/.ssh/authorized_keys` file of app1 and app2 while being `vagrant` user and `root` user separately
- You can change between users with `sudo su - <username>` and while in root, may have to create the .ssh directory and authorized_keys file in it
- After that, doing `sudo ssh -i ansible_id_rsa <username>@<ip of server>` from `~/.ssh` will allow connecting to those machines via ssh after verifying ssh passphrase if set
- Sometimes the host key of the target nodes change giving errors
  - in those cases, we need to go to `.ssh/known_hosts` of our ansible host and empty the file
  - thereafter, use SSH manually to all the target nodes to add the host key to known_hosts again

## Troubleshooting SSH
- Generate keys in `~/.ssh/`
- In `/etc/ssh`, do `sudo vi sshd_config` and make sure you have the following:
  - PubKeyAuthentication yes
  - RSAAuthentication yes
  - PermitRootLogin yes
  - PasswordAuthentication no
- After updating these, make sure to restart ssh service with `sudo systemctl restart ssh.service`
- Every user has its own home directory with `~/.ssh/authorized_keys` so SSH is also specific to that user

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
- For each of the target hosts, we can set a symbolic link as `sudo ln --symbolic /usr/bin/python3 /usr/bin/python` and this way Ansible trying to by default find python2 will be led to python3 and not fail even for the first time (though it will still hang until prompted)
  - one way to prevent this is by running the command with the `-b` flag means privilege escalation with no password prompt while using a passwordless SSH key
  - using `-K` (capital K) will prompt for password to escalate priveleges but after that works for all hosts continuously without hanging
  - the reason why it used to work the second or third time is because the connection is already open

---

## Ansible playbooks
- Create a playbook as shown in project for first_playbook
- Run `ansible-playbook -i inventory.yml playbook.yml` and watch as it completes the tasks
  - it still hangs until prompt because of the python interpreter issue but works
  - the yml file has hyphens which is required for it to work
  - using `-b` flag makes it work all the time without hanging

---

## Setting up docker containers on target machines
- Follow steps at https://docs.docker.com/engine/install/debian/ to install docker on all machines
- specifically for the jenkins node vms (that run the jenkins container), the current user must be in docker group
  - make sure docker group exists and `/var/run/docker.sock` is configured to use that group
  - use `sudo usermod -aG docker vagrant` assuming vagrant as current user to add it to docker group
  - restart the VM for changes to take effect
  - this allows the jenkins container to access docker.sock without permission issues
- For now, we will run ansible directly on ansible control node to distribute the spring boot docker image on the target machines
- install pip on target machines by `sudo apt-get install pip`
- Follow the `python_docker_install_playbook.yml` to install python docker module in target machines
- Follow the `docker_playbook.yml` to get the container on to the target machines
  - pip installs packages on a per-user basis so make sure that the docker_install playbook and the docker playbook, both use the same remote user which is root (due to permission requirements).
  - This pulls the image from dockerhub and attempts to run it
  - In case container randomly exits, check logs with `docker logs <container-id>` or check errors and exit codes via `docker inspect <container-id>`
  - after increasing RAM memory, it works as expected with the containers started

---

## Testing container to VM communication
- Log into jenkins container as root with `sudo docker exec --user="root" -it <container-name> /bin/bash`
- Install vim if not there with `apt-get install vim`
- Go to ~/.ssh and create the .ssh directory if not already there
- Create a new file with vim and copy the private key from host VM into this file, call it `ansible_id_rsa` again
- Update permissions for private key file to be read-writable only by you (else SSH will not use the key) by `chown 600 ansible_id_rsa`
- Run `ssh -i ansible_id_rsa vagrant@192.168.0.105` to ssh into target VM
- With this, we know that installing ansible into the container will still allow communication to the target VMs as long as the private key can be copied in

---

## Ansible variables
- To use variables inside playbooks in a scalable way, we can create a `group_vars` directory in the inventory and playbook directory
- we can create a yml file in group_vars named with one of the host namepaces, like in inventory yml, we have `app`
- In app.yml, we specified a tag value for our spring_boot_docker image and used that in our playbook during deployment
- using this, we can specify what version should be deployed on a particular environment
- while this makes sense for prod env, it doesn't make a lot of sense for staging env where we might want to directly deploy the image that we just built
- so we can basically have a jenkinsfile that will build an image with tag as `snapshot` and then have a separate ansible variable for the staging servers that always use the snapshot tag instead of a specific numeric tag

---

## Ansible Galaxy

- Ansible galaxy is a large public repository of ansible content
- In our playbook, we have used `docker_container` and `docker_prune` modules
- These modules are part of the `community docker` ansible collection
- Ansible collections are a collection of playbooks, plugins, modules and roles
- `docker_container` module allows us to control containers
  - here we just set it to start in detached mode with a port mapping of our choice
- `docker_prune` module allows us to control pruning of any docker element
  - here we use it to prune old images so as to be space-efficient
  - by default, it will only remove dangling images which refers to an image with same name and tag as an existing one but not being used
  - we have to specifically set `images_filter.dangling: false` to remove all unused images including non-dangling ones

---

## Directory Structure

- We create a new directory for each environment and place an inventory file in each with its own group_vars
- The group_vars are picked up from the inventory file directory by default
- The file inside group_vars should mimic the name of some group in the inventory or else it won't be considered
- We create a separate directory called playbooks which houses just the playbooks
- These playbooks are supposed to be independent of environment so that we can choose inventory file dynamically
- Created `app_stage` and `app_prod` groups with a different VM in each
- When we refer to hosts `all` in playbooks, it means all VMs in the inventory
- This allows handling separate environments
  - here we test by building main and snapshot and updating the inventory file to be used in each case
  - snapshot runs stage inventory which will use the snapshot image
  - main runs prod inventory which will use the hardcoded image

---

## Externalizing properties

- add external props as environment props to docker_container using `env` of docker_container module of Ansible
- pass the environment props values from the app_stage and app_prod yml files on a per environment basis
- if an image tag like `snapshot` is being overwritten, the new docker image update for the same tag are never pulled
  - thus, we add the `pull` flag to docker_container in the playbook so that the image is refreshed everytime
  - but we don't want to do that for prod and only for stage as snapshot only exists on stage
  - therefore have a group_var and then use pull flag with that variable
  - container will get updates if the image is refreshed

---

## Externalizing passwords

- Need ansible vault for it so in jenkins container, we need `pip3 install ansible-vault`
- We create a directory structure like `group_vars/{ansible_group}/` with `vars.yml` and `vault.yml` in it
  - `ansible_group` here implies `app_stage`, `app_prod` etc
- we define a key value pair in the vault.yml and encrypt it using `ansible-vault encrypt --vault-id {vaultId}@{password_file} vault.yml`
  - we can keep something like `stage_vault` and `prod_vault` as vault-ids for our two environments
  - we can see the current vaultIds on the first line of the encrypted file at the end
  - reading vault.yml thereafter shows encrypted content, which you can then track in git
- adding a new key to vault yml would require the following:
  - `ansible-vault edit --vault-id {vaultId}@{password_file} vault.yml` which opens a temp version of file decrypted
  - we can add the new key value pair for the password here and save it, and that encrypts the file with the updates
  - we can use `ansible-vault view --vault-id {vaultId}@{password_file} vault.yml` to view the details in terminal
  - but obviously try not to view it as part of automated processes as it is a security risk
- Once this is done, we store the vault passwords in jenkins credentials and use them in the jenkinsfile to decrypt the vaults
- To update passwords and kick off deployment
  - load your encrypted file and use `ansible-vault edit` to edit it
  - then open your file normally with vi to get the encrypted content and push it back into git
  - make sure to remove the password file if you created one for this
  - kick off a build for the dependent applications from jenkins which will redeploy with the changes
- The current vault password is stored in `~/ansible-learning/vault_password_file` on the VM directly for now

---

# Multiple apps setup

- If there are multiple apps to be deployed and we wanted to do it with one playbook
  - Each app will have its own Jenkinsfile which will internally run the common Ansible playbook
  - Have one task for each app and add a tag to each task corresponding to that app
  - Finally, keep a generic cleanup task with a tag of common
  - In the Jenkinsfile, when invoking the playbook command, specify the tag for the app and the common tag to only execute those two

---
