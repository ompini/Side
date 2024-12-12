Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.21.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export SIDE_CHAIN_ID="sidechain-testnet-4"" >> $HOME/.bash_profile
echo "export SIDE_PORT="45"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf side
git clone https://github.com/sideprotocol/side.git
cd side
git checkout v0.9.5
make install
```

**config and init app**
```
sided config node tcp://localhost:${SIDE_PORT}657
sided config keyring-backend os
sided config chain-id sidechain-testnet-4
sided init "test" --chain-id sidechain-testnet-4
```

**download genesis and addrbook**
```
wget -O $HOME/.side/config/genesis.json https://server-5.itrocket.net/testnet/side/genesis.json
wget -O $HOME/.side/config/addrbook.json  https://server-5.itrocket.net/testnet/side/addrbook.json
```

**set seeds and peers**
```
SEEDS="9c14080752bdfa33f4624f83cd155e2d3976e303@side-testnet-seed.itrocket.net:45656"
PEERS="bbbf623474e377664673bde3256fc35a36ba0df1@side-testnet-peer.itrocket.net:45656,7ce65ae96e8b51e668fd4eb3c12b518225a5506f@95.216.243.177:26656,446e6f34c18de1a64d4d53014e5611e31d34466f@65.108.142.199:55656,3c907d62785b8cbcc21fb40f30a8157666a31176@88.198.70.23:26356,0c85cf29bd251fb4cf2c53943e36bd9d305ee27e@161.97.167.196:18656,b60a5456c46eb9d2a079fc88f7b3dd04cd826be5@93.159.130.38:36656,71d7a672c1d89f7f262049be03cbd28bbc58c2d3@45.129.147.247:36656,a2ed5f49a5370f5e7563fbcb471a7e9c0c1e8a98@213.239.198.181:656,be133ebd4d4bc4adfc0b06114a96d581a9290c98@212.90.120.2:26656,99df9c410d7269faf142ab72f1d5d8b22dd2d1ee@14.167.154.230:45656,c64da4e0565ff5651545b53548b933841577e84d@5.161.100.226:26656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.side/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${SIDE_PORT}317%g;
s%:8080%:${SIDE_PORT}080%g;
s%:9090%:${SIDE_PORT}090%g;
s%:9091%:${SIDE_PORT}091%g;
s%:8545%:${SIDE_PORT}545%g;
s%:8546%:${SIDE_PORT}546%g;
s%:6065%:${SIDE_PORT}065%g" $HOME/.side/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${SIDE_PORT}658%g;
s%:26657%:${SIDE_PORT}657%g;
s%:6060%:${SIDE_PORT}060%g;
s%:26656%:${SIDE_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${SIDE_PORT}656\"%;
s%:26660%:${SIDE_PORT}660%g" $HOME/.side/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.side/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.side/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.side/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.005uside"|g' $HOME/.side/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.side/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.side/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/sided.service > /dev/null <<EOF
[Unit]
Description=Side node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.side
ExecStart=$(which sided) start --home $HOME/.side
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
sided tendermint unsafe-reset-all --home $HOME/.side
if curl -s --head curl https://server-5.itrocket.net/testnet/side/side_2024-11-30_890827_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-5.itrocket.net/testnet/side/side_2024-11-30_890827_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.side
    else
  echo "no snapshot found"
fi
```

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable sided
sudo systemctl restart sided && sudo journalctl -u sided -f
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/side/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
sided keys add $WALLET

# to restore exexuting wallet, use the following command
sided keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(sided keys show $WALLET -a)
VALOPER_ADDRESS=$(sided keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
sided status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
sided query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.side/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://side-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, uside
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
sided tx staking create-validator \
--amount 1000000uside \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(sided tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id sidechain-testnet-4 \
--gas auto --gas-adjustment 1.5 --fees 1500uside \
-y
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${SIDE_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop sided
sudo systemctl disable sided
sudo rm -rf /etc/systemd/system/sided.service
sudo rm $(which sided)
sudo rm -rf $HOME/.side
sed -i "/SIDE_/d" $HOME/.bash_profile
