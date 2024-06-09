<h1 align="center">Deploy zk-rollup via Airchains</h1>

> Save all of addresses or private-keys during the installation
>
> Save /root/.tracks folder in your local computer in the end
>
> Use Mobaxterm client to connect to your VPS (it's better than putty or termius)

### Official Links
* [0xMoei Twitter](https://twitter.com/0xMoei)
* [Aircahins Twitter](https://twitter.com/airchains_io)
* [Aircahin Discord](https://discord.gg/airchains)

## System Requirements (Minimum-Recommended)
| Ram | cpu     | disk                      |
| :-------- | :------- | :-------------------------------- |
| `2-4 GB`      | `2-4 Core` | `+200 GB SSD` |

<h1 align="center">Steps</h1>

## Install Dependecies
```console
# Update Packages
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git jq lz4 build-essential cmake perl automake autoconf libtool wget libssl-dev

# Install Go
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.3.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
source .bash_profile
go version
```
```console
# Clone Airchains repositories
git clone https://github.com/airchains-network/evm-station.git
git clone https://github.com/airchains-network/tracks.git
git clone https://github.com/availproject/availup.git
```

## Install Evmos (EVM Station)
```console
# Go to directory
cd evm-station

# Install
go mod tidy
/bin/bash ./scripts/local-setup.sh
```

## Config Evmos systemD
```console
# Create env file
nano ~/.rollup-env
```
> Copy and Paste below codes in the env file (Don't Change anything)
```console
MONIKER="localtestnet"
KEYRING="test"
KEYALGO="eth_secp256k1"
LOGLEVEL="info"
HOMEDIR="$HOME/.evmosd"
TRACE=""
BASEFEE=1000000000
CONFIG=$HOMEDIR/config/config.toml
APP_TOML=$HOMEDIR/config/app.toml
GENESIS=$HOMEDIR/config/genesis.json
TMP_GENESIS=$HOMEDIR/config/tmp_genesis.json
VAL_KEY="mykey"
```

> Now let's start Evmos with systemD
>
> Copy Paste the entire code below in the terminal
>
> We use vps so our user is root but if you use wsl you need to change user
```console
sudo tee /etc/systemd/system/rolld.service > /dev/null << EOF
[Unit]
Description=ZK
After=network.target

[Service]
User=root
EnvironmentFile=/root/.rollup-env
ExecStart=/root/evm-station/build/station-evm start --metrics "" --log_level info --json-rpc.api eth,txpool,personal,net,debug,web3 --chain-id "stationevm_1234-1"
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
```

## Start Evmos
```console
sudo systemctl daemon-reload
sudo systemctl enable rolld
sudo systemctl start rolld
sudo journalctl -u rolld -f --no-hostname -o cat
```
> You should see the logs like below now, To exit: CTRL + C

## Get Private-key of Evmos (EVM Station)
```console
/bin/bash ./scripts/local-keys.sh
```
