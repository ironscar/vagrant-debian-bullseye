# Introduction

## Create & SSH into machine

- first step is downloading virtualbox and vagrant
- second step is downloading the linux box for vagrant ``` vagrant init debian/bullseye64 ```
- there are preconfigured boxes already available
- this creates a vagrantfile with some content
- to start the machine with basic settings, just run ```vagrant up```
- sometimes the boot time takes too long and a timeout occurs but you can start Virtualbox and check whether that vm is running or not
- once running, run ```vagrant ssh``` to connect to it via ssh (sometimes this doesn't work)
  - in this case, halt the machine with ```vagrant halt``` and then open virtualbox
  - open settings for the machine and go to network
  - select advanced options and set adapter type to "T server" and repeat the above steps
- at this point your machine is up and you are connected to it at /home/vagrant directory
- you can run ```logout``` on the machine terminal to jump out of connection

## Install required software

- ```sudo apt get update && sudo apt-get upgrade``` to get latest Debian packages
- ```sudo apt-get autoremove``` to clean up and remove unrequired packages
- ```sudo apt-get install build-essential``` to install gcc, make etc required to compile python
- ```sudo apt-get install zlib1g zlib1g-dev``` to install zlib for decompressing data
- ```sudo apt install libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libsqlite3-dev``` to install additional packages required by python & pip
- ```sudo apt-get install libffi-dev``` for _ctypes module that is required for ansible

## Install Python for Ansible

- ```wget https://www.python.org/downloads/release/python-3812/``` to download the python zipped package (find the appropriate version, this one is 3.8.12)
- ```tar -xf Python-3.8.12-tgz``` to unzip it
- ```sudo mv Python-3.8.12 /opt/Python-3.8.12``` to move the installation to opt directory and then go inside /opt/Python directory
- ```./configure --enable-optimizations --enable-shared``` to verify dependencies and configure the environment
- the above step may specify errors and if successful, creates a makefile
- ```make``` to compile makefile
- ```sudo make altinstall``` to install Python binaries and we use altinstall to not overwrite the default Python3 binaries (this requires sudo even if just "make" does not)
  - above command returned exit code 1 => not sure how to fix
- ```sudo ldconfig /opt/Python-3.8.12``` to configure dynamic linker run-time bindings
- ```python3.8 -V``` (capital V) to print version and you should see Python 3.8.12

## Install Pip

- ```wget https://bootstrap.pypa.io/get-pip.py``` to download pip
- ```python3.8 get-pip.py``` to install pip (may warn about pip not being on path, in which case follow next sub-point)
  - ```export PATH = $HOME/.local/bin:$PATH``` will add the pip install dir to PATH where $HOME will be /home/vagrant and $PATH is the rest of the paths configured
- ```python3.8 -m pip install --upgrade pip``` to upgrade pip version
- ```pip3.8 --version``` to check pip version and that its installed properly

## Install Ansible

- ```pip3 install --user ansible``` to install ansible
  - gave error that "python setup.py egg_info did not run successfully"
  - ```pip3 list``` to confirm setuptools is installed and otherwise run ```pip3 install --user setuptools```
- ```ansible --version``` to check ansible-core version and if its installed successfully
- ```pip3 list``` to check version of ansible and ansible-core
