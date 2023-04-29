# Sparrow Server - 24/7 Mixing

### Introduction
In this guide I will walk you through how to use [Sparrow Server](https://www.sparrowwallet.com) on your headless Bitcoin node to allow you to mix 24/7 without leaving your laptop running all night.

I used [RaspiBolt's Guide](https://raspibolt.org/guide/bonus/bitcoin/sparrow-terminal.html) as the basis for this guide but modified it for a Intel/AMD Debian/Ubuntu based server.

### Pre-requisites
Technically you can do this without a private Electrum server and opt to use the public Electrum servers that Sparrow offers in it's server preferences. If you want to use your own node I used @k3tan's [Ministry of Nodes Guide](https://youtube.com/playlist?list=PLCRbH-IWlcW2A_kpx2XwAMgT0rcZEZ2Cg) 01 to 05 as the basis of my home Electrum server.

Setup a Local machine with +1TB SSD:
- [UNB22 - 01 - Overview](https://youtu.be/9Kb7TobTNPI)
- [UNB22 - 02 - Planning Preparation and Installation of Ubuntu](https://youtu.be/siCQvYD6pro)
- [UNB22 - 03 - Ubuntu Familiarisation](https://youtu.be/YpRuP_X1D2s)

Setup Bitcoin Core on Local machine:
- [UNB22 - 04 - Bitcoin Core](https://youtu.be/fx_mLXISrfM)

Setup Fulcrum Server OR ElectrumX Server on Local machine:
- [UNB22 - 05 - Fulcrum Server](https://youtu.be/SpQRrbJt7cg) OR
- [Running an ElectrumX Server](https://youtu.be/QiX0rR_o_fI)

This guide also assumes you already have Sparrow Wallet files that are setup to use [Whirlpool](https://www.sparrowwallet.com/docs/mixing-whirlpool.html).

### Installing Sparrow Server

SSH into your headless server:
```bash
ssh username@{headless-ip}
sudo apt update && sudo apt upgrade
### Download, verify and install Sparrow Server
VERSION="1.7.6"
wget https://github.com/sparrowwallet/sparrow/releases/download/$VERSION/sparrow-server_$VERSION-1_amd64.deb
wget https://github.com/sparrowwallet/sparrow/releases/download/$VERSION/sparrow-$VERSION-manifest.txt.asc
wget https://github.com/sparrowwallet/sparrow/releases/download/$VERSION/sparrow-$VERSION-manifest.txt
curl https://keybase.io/craigraw/pgp_keys.asc | gpg --import
gpg --verify sparrow-$VERSION-manifest.txt.asc
sha256sum --check sparrow-$VERSION-manifest.txt --ignore-missing
sudo dpkg -i sparrow-server_$VERSION-1_amd64.deb
### Often the location of the Sparrow binary won't be in your $PATH
### Edit your .bashrc (Also install vim because it kicks ass):
sudo apt install vim
sudo vim ~/.bashrc
### Add the end of the .bashrc include this:
export PATH="/opt/sparrow/bin:$PATH"
### exit with :wq
### reload your bash script:
source ~/.bashrc
```

### Keeping Sparrow Server Running 24/7

If you open `Sparrow` from your ssh session you will get a nice blue colored terminal UI. Familiarise yourself with the interface it varies in a few ways from the regular desktop GUI. Exit `Sparrow` in your ssh session.

#### Optional: Connect to your local Electrum Server

If you are running your own Electrum Server on the same headless server then while within `Sparrow` go to `Preferences > Server` and select `Private Electrum` and `Continue`. Set values according to your Electrum Server implementation and test connection.

```bash
# For Electrs (default)
URL: 127.0.0.1:50001
Use SSL?: No
  
# For Fulcrum 
URL: 127.0.0.1:50002
Use SSL?: Yes 
```

You are now connected to your own Electrum Server

#### Copy existing Wallet files to server

Install [rsync](https://www.digitalocean.com/community/tutorials/how-to-use-rsync-to-sync-local-and-remote-directories) on both your laptop and your headless server:
```bash
sudo apt install rsync
ssh username@{headless-ip}
sudo apt install rsync
exit
### copy wallet files to server
rsync -aP .sparrow/wallets/* username@{headless-ip}:.sparrow/wallets/
```

### Keeping Sparrow Server Running 24/7 - ctd.

Sparrow server doesn't come with a way to run as a `system daemon`, so you will have use something like [tmux](https://linuxhandbook.com/tmux/) to allow you to run Sparrow server and keep it running in the background on your server.

While still in your ssh session install tmux:
```bash
sudo apt install tmux
```

Now start a new tmux session for your Sparrow Server
```bash
tmux new -s sparrowserver
### This will open a tmux terminal instance
### Run Sparrow
Sparrow
```

Using Sparrow Server open the wallet files you copied across. Enter passwords and passphrases (if needed). Go to `Postmix > UTXOs > Mix To...` and set `Postmix index range` to Odd. Now lock your wallet files and exit the tmux session by using `ctrl+b` then `d`.

#### Accessing your tmux session
When you want to access the tmux session again use this command:
```bash
tmux a -t sparrowserver
```
This will bring up your Sparrow Server instance as you left it.
