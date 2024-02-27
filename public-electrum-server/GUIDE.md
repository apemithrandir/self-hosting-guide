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

In addition to the above you will also need a remote server:
- [Host4Coins](https://host4coins.net/)
- [1984Hosting](https://1984.hosting/)

### Local Machine Setup

This guide assumes you have a local machine running Debian-based Linux Distro with a fully sync-ed Bitcoin Node and Bitcoin indexer either ElectrumX or Fulcrum (ElectRS is another option ElectrumX is many times faster than this).

The way we are going to expose our Bitcoin indexer to the public is via a [Reverse SSH tunnel](https://youtu.be/N8f5zv9UUMI) from our local machine to a remote server.

This [guide](https://openoms.github.io/bitcoin-tutorials/ssh_tunnel.html) from @openoms covers some of this but not specifically from the perspective of tunnelling your Electrum Server.

You should have [ssh keys setup](https://www.cyberciti.biz/faq/how-to-set-up-ssh-keys-on-linux-unix/) and copied over to your remote server. For this ssh tunnel daemon to work smoothly you will need ssh keys without a passphrase.

First install autossh which is a wrapper on ssh:
```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install autossh
```

Then create a `.service` file to run your ssh tunnel daemon:
```bash
sudo vim /etc/systemd/system/ssh-tunnel.service
```

Here is a template of this `.service` file:
```service
[Unit]
Description=Remote SSH tunnel for Electrum Server
After=network.target

[Service]
User=localuser
Group=localusergroup
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -C -M 0 -v -N -o "ServerAliveInterval=60" -R <remote-port>:localhost:50001 <remote-username>@<remote-ip-or-domain>
Restart=always
RestartSec=60
StandardOutput=journal

[Install]
WantedBy=multi-user.target
```
_Note: Remote port should not be equal to 50001 or 50002 to avoid potential binding issues on your remote server._

The port you are tunneling should be the regular TCP port 50001 and not the SSL
port 50002. This is because the remote server will be performing the SSL
encryption via nginx when exposing the data to the public. You want to edit the config file for your Electrum Server and make sure
the line relating to enabling tcp is uncommented. In `fulcrum.conf` this is
near line ~120 `tcp = 0.0.0.0:50001`. 

Once the daemon file `ssh-tunnel.service` has been created you will need to
reload, enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable ssh-tunnel.service
sudo systemctl start ssh-tunnel.service
```

You should then check the status:
```bash
sudo systemctl status ssh-tunnel.service
```
or logs:
```bash
journalctl -fu ssh-tunnel.service
```

This important line in the logs you should be looking for is this:
```bash
autossh[<process-id>]: debug1: remote forward success for: listen <remote-port>, connect localhost:50001
```

### Remote Server Setup

The remote server should be running a debian-based headless distro. You will need
[nginx
installed](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/).
If you got your server from [1984Hosting](https://1984.hosting/) they have the
option to pre-install some packages including nginx.

As per [@openoms guide](https://openoms.github.io/bitcoin-tutorials/ssh_tunnel.html) you should login as root or run:
```
sudo su
```
edit the sshd config:
```bash
vim /etc/ssh/sshd_config
```
Make sure the following entries are active. You can search for them in the config and remove the # to activate them or if they are not included just paste them on the end of the file:
```
RSAAuthentication yes
PubkeyAuthentication yes
GatewayPorts yes
AllowTcpForwarding yes
ClientAliveInterval 60
```

Restart the sshd service (WARNING: you can lose access at this point if the config is wrong):
```
systemctl restart sshd
```

Log back onto your remote server and check that the reverse ssh-tunnel is working:
```bash
lsof -i :<remote-port>
```
This should return:
```bash
COMMAND     PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
sshd   <pid-v4> root    7u  IPv4 00000000      0t0  TCP *:<remote-port> (LISTEN)
sshd   <pid-v6> root    8u  IPv6 00000000      0t0  TCP *:<remote-port> (LISTEN)
```
You can also use:
```bash
netstat -tulpn | grep <remote-port>
```
which should return:
```bash
tcp        0      0 0.0.0.0:<remote-port>           0.0.0.0:*               LISTEN      <pid-v4>/sshd: <remote-username>  
tcp6       0      0 :::<remote-port>                :::*                    LISTEN      <pid-v6>/sshd: <remote-username>
```

Now you will need to edit your nginx config (use sudo if not logged in as root):
```bash
vim /etc/nginx/nginx.conf
```
Then add this section before the `http{}` part of the config:
```conf
stream {
  server {
     listen [::]:50002 ssl;
     listen 50002 ssl;
     proxy_pass localhost:<remote-port>;
     ssl_certificate /etc/ssl/<remote-ip-or-domain>/server.crt;
     ssl_certificate_key /etc/ssl/<remote-ip-or-domain>/server.key;
     error_log /var/log/nginx/error.log;
  }
}
```
_Note: If you run into issues with stream and get the error `unknown directive "stream" in /etc/nginx/nginx.conf:` after adding the above and running `nginx -t`. Then you should try installing `libnginx-mod-stream` via `apt install libnginx-mod-stream` ([link](https://www.server-world.info/en/note?os=Ubuntu_22.04&p=nginx&f=12))._
#### HTTPS Certificates
*Update: Due to restrictions in BDK you should try to obtain a certificate via Let's Encrypty versus using a self-signed certificate as described below. You can follow this guide if you already have nginx and certbot installed on your VPS: https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04*

Now you might be wondering where to get the `ssl_certificate` and
`ssl_certificate_key`. If you already setup ssl on your Electrum server on your
local machine then you can use
[scp](https://www.freecodecamp.org/news/scp-linux-command-example-how-to-ssh-file-transfer-from-remote-to-local/)
to copy those certificate and keys to your remote server and reuse them.

Otherwise you can create a fresh set of keys (add sudo if not logged in as
root):
```bash
apt install openssl
mkdir /etc/ssl/<remote-ip-or-domain> 
cd /etc/ssl/<remote-ip-or-domain>/
openssl genrsa -des3 -out server.pass.key 2048
openssl rsa -in server.pass.key -out server.key
rm server.pass.key
openssl req -new -key server.key -out server.csr
openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
rm server.csr
```

Now you need to check that you haven't messed up your `nginx.conf` by running:
```bash
nginx -t
```
This should return:
```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Now reload the daemon and restart nginx:
```bash
systemctl daemon-reload
systemctl restart nginx
```
Now you should check the status of nginx:
```bash
systemctl status nginx
```

If you get something like this:
```
nginx: [emerg] bind() to 0.0.0.0:50002 failed (98: Address already in use)
```

Then it means you are re-using one of your ports. Stop nginx and have a look
at:
```
lsof -i :50002
```
with nginx stopped there shouldn't be anything running on your remote server
over that port. If there is then you might need to change the listen port in your
stream nginx config.

Now in order for someone to use your public facing Electrum server they will
need to enter use `<remote-ip-or-domain>:50002`. This means that you will need
to open traffic over port 50002:
```bash
apt install ufw
ufw status
ufw allow 50002
ufw status
```
You will also want to look into some basic server security:
  - [How to disable ssh password login](https://www.cyberciti.biz/faq/how-to-disable-ssh-password-login-on-linux/)
  - [Fail2Ban](https://github.com/fail2ban/fail2ban)

### Acknowledgements
Thanks to [wiz](https://github.com/wiz) and [emzy](https://github.com/Emzy) for helping me when I was setting this up for myself.

### Issues

If you need help with this guide you can create an issue on the repo and I will help you there.
