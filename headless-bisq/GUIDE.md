# Headless Bisq Instance

### Introduction

In this guide I will walk you through how to run the [Bisq GUI](https://bisq.network/) on your headless server. If like me you love using Bisq but don't like keeping a laptop open running the GUI 24/7 to keep your offers in the orderbook then this is the guide for you.

### Pre-requisites

You will need a headless server running Bitcoin Core in order to make this work. I used @k3tan's [Ministry of Nodes Guide](https://youtube.com/playlist?list=PLCRbH-IWlcW2A_kpx2XwAMgT0rcZEZ2Cg) 01 to 04 as the basis of my Bitcoin Core Server.

Setup a Local machine with +1TB SSD:
- [UNB22 - 01 - Overview](https://youtu.be/9Kb7TobTNPI)
- [UNB22 - 02 - Planning Preparation and Installation of Ubuntu](https://youtu.be/siCQvYD6pro)
- [UNB22 - 03 - Ubuntu Familiarisation](https://youtu.be/YpRuP_X1D2s)

Setup Bitcoin Core on Local machine:
- [UNB22 - 04 - Bitcoin Core](https://youtu.be/fx_mLXISrfM) 

You will also need a Laptop (or other Computer with display functionality) in order to interact with your headless instance of Bisq.

### First Backup Bisq!

If you have already got a Bisq instance running on your Laptop with your wallet and payment accounts attached to it then you should [Backup your Bisq Data Directory](https://bisq.wiki/Backing_up_application_data). Basically you should close your Bisq instance and then copy the Bisq data directory to your headless server:

```bash
### make sure rsync is installed on both machines
sudo apt install rsync
ssh username@{headless-ip}
sudo apt install rsync
exit
### then sync up your folder to a backup directory
rsync -aP ~/.local/share/Bisq/* username@{headless-ip}:~/BisqBackup_YYYYMMDD/
```

### Installing Bisq

Now you will need to install Bisq on your headless server. SSH into your headless server:

```bash
ssh username@{headless-ip}
sudo apt update && sudo apt upgrade
### Install GUI utility
sudo apt install xdg-utils
### Download, verify and install Bisq
VERSION="1.9.9"
wget https://bisq.network/downloads/v$VERSION/Bisq-64bit-$VERSION.deb
wget https://bisq.network/downloads/v$VERSION/Bisq-64bit-$VERSION.deb.asc
curl https://bisq.network/pubkey/E222AA02.asc | gpg --import
gpg --verify Bisq-64bit-$VERSION.deb.asc
sudo mkdir /usr/share/desktop-directories/
sudo dpkg -i Bisq-64bit-$VERSION.deb
### Often the location of the Bisq binary won't be in your $PATH
### Edit your .bashrc (Also install vim because it kicks ass):
sudo apt install vim
sudo vim ~/.bashrc
### Add the end of the .bashrc include this:
export PATH="/opt/bisq/bin:$PATH"
### exit with :wq
### reload your bash script:
source ~/.bashrc
```

### Headless GUI Interface

The _normal_ way for accessing GUI instances from a headless server is via [X11 forwarding](https://nulb.app/x4mxj). This works but you won't be able to keep your Bisq instance running in the background of your headless server as desired. Enter [XPRA](https://www.xpra.org/). This is software that allows you to keep that GUI instance running even after you stop accessing it from your headsup display.

XPRA must be installed on both the client and the host. Please follow instructions on their [GitHub](https://github.com/Xpra-org/xpra) for installation. I will give instructions for Ubuntu 22.04LTS:

```bash
DISTRO=jammy
#install https support for apt (which may be installed already):
sudo apt update
sudo apt install apt-transport-https software-properties-common
sudo apt install ca-certificates
# add xpra GPG key:
sudo wget -O "/usr/share/keyrings/xpra.asc" https://xpra.org/gpg.asc
# add the xpra repository:
wget -O "/etc/apt/sources.list.d/xpra.sources" https://xpra.org/repos/$DISTRO/xpra.sources
# add the optional beta channel:
# wget -O "/etc/apt/sources.list.d/xpra-beta.sources" https://xpra.org/repos/$DISTRO/xpra-beta.sources
# install the xpra package:
sudo apt update
sudo apt install xpra
```

### Getting your Headless Bisq Initialized

Before you use XPRA to attach a headless GUI version of Bisq you might need to open Bisq using X11 forwarding just to initialize the Bisq data directory folders on your headless server. Log into your server via ssh and edit your your `sshd_config` file to switch any `#X11Forwarding no` to uncommented `X11Forwarding yes`. Also perform `sudo systemctl restart sshd` and/or `sudo systemctl restart ssh`.

Login to your server again using trusted ssh login:
```bash
ssh -Y username@{headless-ip}
### In your server now
Bisq
```

This should now open Bisq GUI on your host machine. Let it boot up and sync. Then once it opens correctly and the app looks in a good state then exit the application. This should have created the directory and also hopefully give you a chance to troubleshoot any Bisq issues you might have before you copy over your Bisq data directory and setup `XPRA`.

Assuming it is all setup well then let us get that data directory that you copied to your server in the _First Backup Bisq!_ section.
```bash
### Delete the files stored during the initialization of Bisq above:
rm -r ~/.local/share/Bisq/*
### copy from your backup folder:
sudo apt install rsync
rsync -aP ~/BisqBackup_YYYYMMDD/* ~/.local/share/Bisq/
```
Now open Bisq on your headless server again and check that your data directory is restored correctly and that you have your wallet and payment accounts preserved from the old Bisq instance.

### XPRA FTW

Now assuming you got XPRA installed on both client and host. First let us ssh into the host and attach Bisq as an XPRA instance:
```
### You can replace :10 without whatever you want just remember it for later
xpra start :10 --start=Bisq
xpra list
```
Now head exit from your headless client and get back into terminal for your host and run this:
```
### Reminder the :10 must match the number used in your headless client above
xpra attach ssh:username@{headless-ip}:10
```

This should hopefully make your Bisq instance pop up on the host machine in all it's glory. You can now create an offer and then keep that offer running on the server and shudown your host machine. Just use `CTRL+C` in the terminal window where you ran `xpra attach ssh:username@{headless-ip}:10`. Happy trading!

### Troubleshooting
#### XPRA folder Permissions
If you run into weird permissions errors around where `xpra` wants to keep it's logs on your headless server then you might need to do this:
```bash
### I'm not very sure about these steps, so if you spot an error let me know:
sudo mkdir /run/user/1000
sudo chown {username} /run/user/1000
sudo mkdir /run/user/1000/xpra
sudo chown {username} /run/user/1000/xpra/
```
#### Usability Quirks
Small quirks to be aware of, you won't easily be able to copy and paste from the XPRA Bisq GUI to your host machine (or at least I haven't figured that out yet). This can make paying from an external wallet a bit cumbersome. Similarly the open in external wallet link _won't work_, in the sense that it is trying to open a wallet on your server. Now maybe you could also run Sparrow Wallet via XPRA and that would work but I haven't tested that yet.

#### Bisq Double Instances
Bisq doesn't especially like if you run multiple instances of the same data directory, so if for some reason you want to stop running the server instance and go back to your Laptop instance I would recommend the following:
```bash
### Open your XPRA instance on your host
xpra attach ssh:username@{headless-ip}:10
### Now rather than CTRL+C escape you must actually shutdown the GUI properly first
### then CTRL+C escape from the xpra attachment.
### Now enter your server:
ssh username@{headless-ip}
### Check XPRA is still running
xpra list
### Stop XPRA for that Bisq process running at :10
xpra stop :10
### It is good practice to backup your current Bisq data directory folder on your server first:
rsync -aP ~/.local/share/Bisq/ ~/BisqBackup_YYYYMMDD
### Now exit your server
exit
### sync up your host machine's Bisq directory with your servers:
rsync -aP username@{headless-ip}:~/.local/share/Bisq/ ~/.local/share/Bisq/
```

Now you should be able to open your Bisq on your host machine without causing you any issues with open offers and trades. Though I would advise against doing this when you are in the middle of a trade settlement just to err on the side of caution.

Similarily if you are now going back to your server instance you should close that GUI instance down and now copy your host machine's Bisq directory back to your server:
```bash
### It is good practice to backup your current Bisq data directory folder on your server first:
rsync -aP ~/.local/share/Bisq/ ~/BisqBackup_YYYYMMDD
### sync up your server's Bisq directory with your host machine:
rsync -aP ~/.local/share/Bisq/ username@{headless-ip}:~/.local/share/Bisq/
ssh username@{headless-ip}
xpra start :10 --start=Bisq
xpra list
exit
xpra attach ssh:username@{headless-ip}:10
```
