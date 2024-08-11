# Phoenix Server - Easy Lightning Node

## Introduction
In this guide I will walk you through how to install [Phoenix Server AKA PhoenixD](https://github.com/ACINQ/phoenixd) on your headless server to give you the functionality of Phoenix LN Mobile wallet but on your server.

## Installing PhoenixD

Grab the install files from the latest release: https://github.com/ACINQ/phoenixd/releases
```bash
VERSION="0.3.2"
wget "https://github.com/ACINQ/phoenixd/releases/download/v${VERSION}/phoenix-${VERSION}-linux-x64.zip"
wget "https://github.com/ACINQ/phoenixd/releases/download/v${VERSION}/SHA256SUM.asc"
wget https://acinq.co/pgp/drouinf.asc
gpg --import drouinf.asc
gpg -d SHA256SUM.asc > SHA256SUM.stripped
sha256sum -c SHA256SUM.stripped --ignore-missing
```

Assuming the checksum and signature are valid you can then proceed with copying the binary files to your binary directories.
```bash
VERSION="0.3.2"
sudo rsync -aP "phoenix-${VERSION}-linux-x64/*" /usr/local/bin/
sudo rsync -aP "phoenix-${VERSION}-linux-x64/*" /var/lib/
```

## Setting up PhoenixD

The setup process is very easy. Just go into your commandline and run `phoenixd`. This will initialize a folder called `~/.phoenix` which contains all the relevant files to run your PhoenixD.

After you run `phoenixd` for the first time, interrupt the command with `CTRL+C` and this will shut it down. Now you just need to setup a `.service` file to make sure it remains running in the background 24/7.

### Backup Seed

The 12-word seed phrase for your wallet is stored in plain text in `~/.phoenix/seed.dat`. Back up this phrase somewhere for recovery. I would probably recommend backing up the complete contents of `~/.phoenix`.

### Review Defaults
You might also want to review some of the default settings and decide whether you want to override these in `~/.phoenix/phoenix.conf`. In particular you might want to set a different max mining fee.
```bash
Liquidity Options:
  --auto-liquidity=(off|2m|5m|10m)  Amount automatically requested when inbound liquidity is needed (default:
                                    2m)
  --max-mining-fee=<int>            Max mining fee for on-chain operations, in satoshis (default: 1% of
                                    auto-liquidity amount)
  --max-fee-credit=(off|50k|100k)   Max fee credit, if reached payments will be rejected (default: 100k)

Options:
  --chain=(mainnet|testnet)              Bitcoin chain to use (default: mainnet)
  --mempool-space-url=<value>            Custom mempool.space instance
  --http-bind-ip=<text>                  Bind ip for the http api (default: 127.0.0.1)
  --http-bind-port=<int>                 Bind port for the http api (default: 9740)
  --http-password=<text>                 Password for the http api (full access)
  --http-password-limited-access=<text>  Password for the http api (limited access)
  --webhook=<value>                      Webhook http endpoint for push notifications (alternative to
                                         websocket)
  --webhook-secret=<text>                Secret used to authenticate webhook calls
  --silent, --verbose                    Verbosity level (default: prints high-level info to the console)
```

### Create .service file

```bash
sudo vim /etc/systemd/system/phoenixd.service
```

```bash
Unit]
Description=PhoenixD
After=network.target

[Service]
ExecStart=/usr/local/bin/phoenixd
User={your_username}

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable phoenixd.service
sudo systemctl start phoenixd.service
sudo systemctl status phoenixd.service
```

## Using PhoenixD

This new Phoenix instance will charge around 1% fee + mining costs, but it manages all the liquidity for you. You can LN receive payments immediately and I don't yet see a native option for sending from onchain to the wallet as you might in Phoenix mobile.

To create a channel you run `phoenix-cli getoffer` and pay from another LN wallet to the static invoce. Once you do this you can do `phoenix-cli getinfo` and `phoenix-cli getbalance` to monitor your wallet.

Read more about the Auto-Liquity setup [here.](https://phoenix.acinq.co/server/auto-liquidity)

## Bonus Setup - Phoenixd-Server-Ui

Hodladi has setup a web-ui for interfacing with your PhoenixD instance:
https://github.com/Hodladi/Phoenixd-Server-Ui

You can self-host this or use his public instance: https://pwallet.app/

If you use the public instance you will have to use something like [CloudFlareD Tunnels](https://www.cloudflare.com/products/tunnel/) to forward your LAN PhoenixD http web socket located at http://localhost:9740 to your own public facing domain name.

If you use a self-hosted instance then you just give it the LAN IP address in full with http:// and the port 9740 together with the `http-password` from `~/.phoenix/phoenix.conf`.

