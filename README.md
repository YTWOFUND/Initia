# Initia

### Initia node Installation Instructions.

[Official documentation](https://docs.initia.xyz)

System requirements:</br>
CPU: Quad Core or larger AMD or Intel (amd64) CPU
RAM:32GB RAM
SSD:1TB NVMe Storage
100MBps bidirectional internet connection
OS: Ubuntu 20.04 or 22.04</br>

You can take a weaker server

### Network configurations: </br>
Outbound - allow all traffic </br>
Inbound - open the following ports :</br>
1317 - REST </br>
26657 - TENDERMINT_RP </br>
26656 - Cosmos </br>
Running as a Provider? Add these specific ports: </br>
22221 - provider port </br>
22231 - provider port </br>
22241 - provider port </br>

### Installing the Babylon Node

1. Preparing the server/Required packages installation</br>
```
sudo apt update
sudo apt upgrade -y
sudo apt-get install libclang-dev
sudo apt install -y unzip logrotate git jq sed wget curl coreutils systemd
```
### Go installation.
```
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.21.10.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
go version
```

# Set vars
```
MONIKER="YOUR_MONIKER_GOES_HERE"
```

### Download and build binaries
```
cd $HOME
rm -rf initia
git clone https://github.com/initia-labs/initia.git
cd initia
git checkout v0.2.14
make build
mkdir -p $HOME/.initia/cosmovisor/genesis/bin
mv build/initiad $HOME/.initia/cosmovisor/genesis/bin/
rm -rf build
sudo ln -s $HOME/.initia/cosmovisor/genesis $HOME/.initia/cosmovisor/current -f
sudo ln -s $HOME/.initia/cosmovisor/current/bin/initiad /usr/local/bin/initiad -f
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```

# Config and init app
```
initiad config set client chain-id initiation-1
initiad config set client keyring-backend test
initiad config set client node tcp://localhost:17957
initiad init $MONIKER --chain-id initiation-1
```

# Download genesis and addrbook
```
curl -Ls https://snapshots.kjnodes.com/initia-testnet/genesis.json > $HOME/.initia/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/initia-testnet/addrbook.json > $HOME/.initia/config/addrbook.json
```

# Set seeds and peers
```
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@initia-testnet.rpc.kjnodes.com:17959\"|" $HOME/.initia/config/config.toml
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.15uinit,0.01uusdc\"|" $HOME/.initia/config/app.toml
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.initia/config/app.toml
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:17958\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:17957\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:17960\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:17956\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":17966\"%" $HOME/.initia/config/config.toml
sed -i -e "s%^address = \"tcp://localhost:1317\"%address = \"tcp://0.0.0.0:17917\"%; s%^address = \":8080\"%address = \":17980\"%; s%^address = \"localhost:9090\"%address = \"0.0.0.0:17990\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:17991\"%; s%:8545%:17945%; s%:8546%:17946%; s%:6065%:17965%" $HOME/.initia/config/app.toml
```

# Create service file
```
sudo tee /etc/systemd/system/initia.service > /dev/null << EOF
[Unit]
Description=initia node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.initia"
Environment="DAEMON_NAME=initiad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.initia/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable initia.service
```

# Reset and download snapshot
```
curl -L https://snapshots.kjnodes.com/initia-testnet/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.initia
[[ -f $HOME/.initia/data/upgrade-info.json ]] && cp $HOME/.initia/data/upgrade-info.json $HOME/.initia/cosmovisor/genesis/upgrade-info.json
```

# enable and start service
```
sudo systemctl start initia.service && sudo journalctl -u initia.service -f --no-hostname -o cat
```

### Becoming a Validator

# Create wallet key new
```
initiad keys add wallet
```

(OPTIONAL) RECOVER EXISTING KEY
```
initiad keys add wallet --recover
```

# check sync status, once your node is fully synced, the output from above will print "false"
```
initiad status 2>&1 | jq .SyncInfo
```

### We receive tokens from the [faucet](https://faucet.testnet.initia.xyz/)

# before creating a validator, you need to fund your wallet and check balance
```
initiad q bank balances $(initiad keys show wallet -a)
```
# Create validator
```
initiad tx mstaking create-validator \
--amount 1000000uinit \
--pubkey $(initiad tendermint show-validator) \
--moniker "YOUR_MONIKER_NAME" \
--identity "FFB0AA51A2DF5955" \
--details "I love YTWO❤️" \
--chain-id initiation-1 \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.05 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.15uinit \
-y
```

### Update
```
No update

Current network:initiation-1
Current version:v0.2.14
```

### Useful commands

Check balance
```
initiad q bank balances $(initiad keys show wallet -a)
```

CHECK SERVICE LOGS
```
sudo journalctl -u initia.service -f --no-hostname -o cat
```

RESTART SERVICE
```
sudo systemctl restart initia.service
```

GET VALIDATOR INFO
```
initiad status 2>&1 | jq .ValidatorInfo
```

DELEGATE TOKENS TO YOURSELF
```
initiad tx mstaking delegate $(initiad keys show wallet --bech val -a) 1000000uinit --from wallet --chain-id initiation-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.15uinit -y
```

REMOVE NODE
Please, before proceeding with the next step! All chain data will be lost! Make sure you have backed up your priv_validator_key.json!
```
cd $HOME
sudo systemctl stop initia.service
sudo systemctl disable initia.service
sudo rm /etc/systemd/system/initia.service
sudo systemctl daemon-reload
rm -f $(which initiad)
rm -rf $HOME/.initia
rm -rf $HOME/initia
```
