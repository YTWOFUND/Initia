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
sudo apt install -y curl git jq lz4 build-essential
```
### Go installation.
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.21.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
source .bash_profile
```

### Download and build binaries
```
cd && rm -rf initia
git clone https://github.com/initia-labs/initia
cd initia
git checkout v0.2.14
make install
```

# Config and init app
```
initiad init "your moniker" --chain-id initiation-1
```

# Download genesis and addrbook
```
curl -L https://snapshots-testnet.nodejumper.io/initia-testnet/genesis.json > $HOME/.initia/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/initia-testnet/addrbook.json > $HOME/.initia/config/addrbook.json
```

# Set seeds and peers
```
SEEDS="cd69bcb00a6ecc1ba2b4a3465de4d4dd3e0a3db1@initia-testnet-seed.itrocket.net:51656"
PEERS="aee7083ab11910ba3f1b8126d1b3728f13f54943@initia-testnet-peer.itrocket.net:11656,a8f3e2d4197b34c11228809d0f785a952905b262@43.131.12.180:26656,429f7db154bb139ab3f8f2a8760914e255337a0f@150.109.235.74:26656,767fdcfdb0998209834b929c59a2b57d474cc496@207.148.114.112:26656,9c0417a610846b3a7fd27ac3afbf3b52b527807c@43.157.82.7:26656,633775ca828f8fc7f5c689a8c950664e7f198223@184.174.32.188:26656,9f0ae0790fae9a2d327d8d6fe767b73eb8aa5c48@176.126.87.65:22656,7317b8c930c52a8183590166a7b5c3599f40d4db@185.187.170.186:26656,35e4b461b38107751450af25e03f5a61e7aa0189@43.133.229.136:26656,6a64518146b8c902ef5930dfba00fe61a15ec176@43.133.44.152:26656,a45314423c15f024ff850fad7bd031168d937931@162.62.219.188:26656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.initia/config/config.toml
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.initia/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.initia/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.initia/config/app.toml
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.15uinit,0.01uusdc"|g' $HOME/.initia/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.initia/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.initia/config/config.toml
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
initiad tendermint unsafe-reset-all --home $HOME/.initia
if curl -s --head curl https://testnet-files.itrocket.net/initia/snap_initia.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://testnet-files.itrocket.net/initia/snap_initia.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.initia
    else
  echo no have snap
fi
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
