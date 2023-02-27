# Public Bitcoin Electrum Server

### Introduction
In this guide I will walk you through how you make your locally hosted Bitcoin Electrum Server accessible via the public domain without exposing your home IP address.

### Pre-requisites
I used @k3tan's [Ministry of Nodes Guide](https://youtube.com/playlist?list=PLCRbH-IWlcW2A_kpx2XwAMgT0rcZEZ2Cg) 01 to 05 as the basis of my setup

Setup a Local machine with +1TB SSD:
- [UNB22 - 01 - Overview](https://youtu.be/9Kb7TobTNPI)
- [UNB22 - 02 - Planning Preparation and Installation of Ubuntu](https://youtu.be/siCQvYD6pro)
- [UNB22 - 03 - Ubuntu Familiarisation](https://youtu.be/YpRuP_X1D2s)

Setup Bitcoin Core on Local machine:
- [UNB22 - 04 - Bitcoin Core](https://youtu.be/fx_mLXISrfM) 

Setup Fulcrum Server OR ElectrumX Server on Local machine:
- [UNB22 - 05 - Fulcrum Server](https://youtu.be/SpQRrbJt7cg) OR
- [Running an ElectrumX Server](https://youtu.be/QiX0rR_o_fI)

In addition to the above you will also need a VPS:
- [Host4Coins](https://host4coins.net/)
- [1984Hosting](https://1984.hosting/)

### Local Machine Setup

This guide assumes you have a local machine running Debian-based Linux Distro with a fully sync-ed Bitcoin Node and Bitcoin indexer either ElectrumX or Fulcrum (ElectRS is another option ElectrumX is many times faster than this).

The way we are going to expose our Bitcoin indexer to the public is via a [Reverse SSH tunnel](https://youtu.be/N8f5zv9UUMI) from our local machine to a VPS.

This [guide](https://openoms.github.io/bitcoin-tutorials/ssh_tunnel.html) from @openoms covers some of this but not specifically from the perspective of tunnelling your Electrum Server.

You should have [ssh keys setup](https://www.cyberciti.biz/faq/how-to-set-up-ssh-keys-on-linux-unix/) and copied over to your VPS. For this ssh tunnel daemon to work smoothly you will need ssh keys without a passphrase.

First install autossh which is a wrapper on ssh:
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install autossh
```

Then create a `.service` file to run your ssh tunnel daemon:
```
sudo vim /etc/systemd/system/ssh-tunnel.service
```

Here is a template of this `.service` file:
```unit
[Unit]
Description=Remote SSH tunnel for multiple TCP applications
After=network.target

[Service]
User=statue
Group=statue
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -C -M 0 -v -N -o "ServerAliveInterval=60" -R <remote_port>:localhost:50001 <VPS-username>@<VPS-ip-or-domain>
Restart=always
RestartSec=60
StandardOutput=journal

[Install]
WantedBy=multi-user.target
```

The port you are tunneling should be the regular TCP port 50001 and not the SSL
port 50002. This is because on the VPS we will be using your cert and key from
your Electrum server to apply SSL via nginx when exposing the data to the
public.
