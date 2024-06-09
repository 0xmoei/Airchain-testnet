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

## Change Ports
> No need to change anything in the commands, as I am just teaching you what's happening with the commands

* we set the variable G_PORT to a favorite number like 17 with these commands
```console
echo "export G_PORT="17"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
* We replace 17 with the first of the ports in app.toml with this command
```console
sed -i.bak -e "s%:1317%:${G_PORT}317%g;
s%:8080%:${G_PORT}080%g;
s%:9090%:${G_PORT}090%g;
s%:9091%:${G_PORT}091%g;
s%:8545%:${G_PORT}545%g;
s%:8546%:${G_PORT}546%g;
s%:6065%:${G_PORT}065%g" $HOME/.evmosd/config/app.toml
```
* We replace 17 with the first of the ports in config.toml with this command
```console
sed -i.bak -e "s%:26658%:${G_PORT}658%g;
s%:26657%:${G_PORT}657%g;
s%:6060%:${G_PORT}060%g;
s%:26656%:${G_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${G_PORT}656\"%;
s%:26660%:${G_PORT}660%g" $HOME/.evmosd/config/config.toml
```
```console
sed -i -e 's/address = "127.0.0.1:17545"/address = "0.0.0.0:17545"/' -e 's/ws-address = "127.0.0.1:17546"/address = "0.0.0.0:17546"/' $HOME/.evmosd/config/app.toml
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
> Get your address in subwallet and get Avail faucet with `$faucet` in the [discord](https://discord.gg/airchains)
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
* Replace `moniker-name` with your favorite name
* I have choosen 17 in `Change Ports` step so I put 17 for the ports here, if you didn't change then you are okay
```console
go run cmd/main.go init --daRpc "http://127.0.0.1:7000" --daKey "Avail-Wallet-Address" --daType "avail" --moniker "moniker-name" --stationRpc "http://127.0.0.1:17545" --stationAPI "http://127.0.0.1:17545" --stationType "evm"
```
![image](https://github.com/0xmoei/rollapp-testnet/assets/90371338/3b92775f-4b21-4097-b78b-b14cd0dcbab6)


## Create Tracks Address
* Replace `moniker-name`
```console
go run cmd/main.go keys junction --accountName moniker-name --accountPath $HOME/.tracks/junction-accounts/keys
```
> Save the output of this command (Mnemonic & Address)
>
> Use wallet address with perfix `air` and get faucet with `$faucet` in `switchyard-faucet` in the [discord](https://discord.gg/airchains)
![Screenshot_44](https://github.com/0xmoei/rollapp-testnet/assets/90371338/c7093cd8-dc19-4f9a-849a-d600fc2affff)

## Run Prover
```console
go run cmd/main.go prover v1EVM
```
![image](https://github.com/0xmoei/rollapp-testnet/assets/90371338/a0d7ed7f-452d-4ea9-b8ba-ba44fd09ae6c)

## Find node_id
* Find node_id with this command and save it
```console
cat ~/.tracks/config/sequencer.toml
```
![Screenshot_45](https://github.com/0xmoei/rollapp-testnet/assets/90371338/b779d597-f793-4f3a-ad31-d8e2a0df709a)

## Create Station
* Replace `moniker-name`
* Replace `WALLET_ADDRESS` with air... wallet you saved before
* Replace `IP` with your VPS server IP
* Replace `node_id` with your node id you saved
```console
go run cmd/main.go create-station --accountName moniker-name --accountPath $HOME/.tracks/junction-accounts/keys --jsonRPC "https://airchains-testnet-rpc.cosmonautstakes.com/" --info "EVM Track" --tracks WALLET_ADDRESS --bootstrapNode "/ip4/IP/tcp/2300/p2p/node_id"
```
![Screenshot_46](https://github.com/0xmoei/rollapp-testnet/assets/90371338/bb5eb443-6d7f-4767-8e29-a9acac313735)

## Create Station systemD file
```console
sudo tee /etc/systemd/system/stationd.service > /dev/null << EOF
[Unit]
Description=station track service
After=network-online.target
[Service]
User=root
WorkingDirectory=/root/tracks/
ExecStart=/usr/local/go/bin/go run cmd/main.go start
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

## Run Station Node with systemD
```console
sudo systemctl daemon-reload
sudo systemctl enable stationd
sudo systemctl restart stationd
sudo journalctl -u stationd -f --no-hostname -o cat
```
![Screenshot_47](https://github.com/0xmoei/rollapp-testnet/assets/90371338/d44ada54-5ba9-4c46-b302-35e1ea0c6acf)
Exit: `Ctrl+C`

## Check Pod Tracker Logs
```console
sudo journalctl -u stationd -f --no-hostname -o cat
```

<h1 align="center">Installation is Complete but How to earn Points?</h1>
You have completed the installation process. You can import your tracker air... wallet mnemonic to Leap Wallet and connect https://points.airchains.io/ to check your points

Yes, You have 0 Points now. The reason for this is that you need to extract a pod to earn points

Each pod is 25 transactions. Each set of 25 transactions will generate 1 pod, and you will earn 5 points from these transactions

The initial 100 points from the installation will become active after the first pod too

You
