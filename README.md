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

# Install Dependecies
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
```

# Install Evmos (EVM Station)
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
> We use root user of vps but if you use wsl or a custom user you must change user by replacing `root` in the code
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
![image](https://github.com/0xmoei/rollapp-testnet/assets/90371338/796b6ed1-6bb4-40d1-babf-9030a548badb)


## Get Private-key of Evmos (EVM Station)
> Save the private key!
```console
/bin/bash ./scripts/local-keys.sh
```

## Change the ports
> if you are running other nodes in your system, they might conflict
>
> it's better to change the port

* we set the variable G_PORT to a favorite nubmer like 
```console
echo "export G_PORT="17"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
* Replace 17 with the first of the ports in app.toml
```console
sed -i.bak -e "s%:1317%:${G_PORT}317%g;
s%:8080%:${G_PORT}080%g;
s%:9090%:${G_PORT}090%g;
s%:9091%:${G_PORT}091%g;
s%:8545%:${G_PORT}545%g;
s%:8546%:${G_PORT}546%g;
s%:6065%:${G_PORT}065%g" $HOME/.evmosd/config/app.toml
```
* Replace 17 with the first of the ports in config.toml
```console
sed -i.bak -e "s%:26658%:${G_PORT}658%g;
s%:26657%:${G_PORT}657%g;
s%:6060%:${G_PORT}060%g;
s%:26656%:${G_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${G_PORT}656\"%;
s%:26660%:${G_PORT}660%g" $HOME/.evmosd/config/config.toml
```
* Restart Evmos systemD
```console
sudo systemctl restart rolld
```
* Check logs if you want
```console
sudo journalctl -u rolld -f --no-hostname -o cat
```

# Install Avail DA
We will use Avail Turing as the DA layer. We have other options like Celestia, EigenLayer or MockDA but we choose Avail, Remember, You CANNOT change DA later

#

```console
cd $HOME
git clone https://github.com/availproject/availup.git
cd availup
/bin/bash availup.sh --network "turing" --app_id 36
```
![image](https://github.com/0xmoei/rollapp-testnet/assets/90371338/1783ce07-86af-44ab-81b8-5edfba31e28e)

* Close it with `Ctrl+C`

## Config systemD for Avail DA
> Copy Paste All the code below in the terminal and Enter!
```console
sudo tee /etc/systemd/system/availd.service > /dev/null <<'EOF'
[Unit]
Description=Avail Light Node
After=network.target
StartLimitIntervalSec=0

[Service]
User=root
Type=simple
Restart=always
RestartSec=120
ExecStart=/root/.avail/turing/bin/avail-light --network turing --app-id 36 --identity /root/.avail/identity/identity.toml

[Install]
WantedBy=multi-user.target
EOF
```
> We use root user of vps but if you use wsl or a custom user you must change user by replacing `root` in the code

## Start Avail DA systemD
```console
systemctl daemon-reload 
sudo systemctl enable availd
sudo systemctl start availd
sudo journalctl -u availd -f --no-hostname -o cat
```
![image](https://github.com/0xmoei/rollapp-testnet/assets/90371338/4180722d-f0a3-4bcd-938a-d85e20106565)
Exit: `Ctrl+C`

## Save Avail DA Seed Phrase (Mnemonic)
```console
cat ~/.avail/identity/identity.toml
```

## Get Faucet
> Import your Avail DA Mnemonic to the [Subwallet](https://www.subwallet.app/download.html) to create a `polkadot` wallet
>
> Get your address in subwallet and get Avail faucet in the [discord](https://discord.gg/airchains)
![Screenshot_43](https://github.com/0xmoei/rollapp-testnet/assets/90371338/2d43b453-9932-4d69-9960-b03b523471ac)
>
> You can also get Avail faucet [here](https://faucet.avail.tools/) (Turing)

# Install Tracks
## Go to tracks directory
```console
cd $HOME
cd tracks
go mod tidy
```

## Initiate Tracks
* Replace `Avail-Wallet-Address` with your Avail DA wallet
* Replace `moniker` with your favorite name
```console
go run cmd/main.go init --daRpc "http://127.0.0.1:7000" --daKey "Avail-Wallet-Address" --daType "avail" --moniker "moniker" --stationRpc "http://127.0.0.1:8545" --stationAPI "http://127.0.0.1:8545" --stationType "evm"
```

## Create Tracks Address
* Replace `moniker`
```console
go run cmd/main.go keys junction --accountName moniker --accountPath $HOME/.tracks/junction-accounts/keys
```
> Save the output of this command
>
> Use the wallet address with perfix `air` and get faucet in `switchyard-faucet` in the [discord](https://discord.gg/airchains)
