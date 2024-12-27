Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 16GB RAM, 300GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```


**install go, if needed**
```
cd $HOME
VER="1.19.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

*set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export ZETACHAIN_CHAIN_ID="athens_7001-1"" >> $HOME/.bash_profile
echo "export ZETACHAIN_PORT="14"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
mkdir -p $HOME/.zetacored/config
cd $HOME
wget -O $HOME/zetacored https://github.com/zeta-chain/node/releases/download/v23.0.0/zetacored-linux-amd64
chmod +x $HOME/zetacored 
mv $HOME/zetacored  $HOME/go/bin
```

**config and init app**
```
zetacored config node tcp://localhost:${ZETACHAIN_PORT}657
zetacored config keyring-backend os
zetacored config chain-id athens_7001-1
zetacored init "test" --chain-id athens_7001-1
```

**download genesis and addrbook**
```
wget -O $HOME/.zetacored/config/genesis.json https://server-4.itrocket.net/testnet/zetachain/genesis.json
wget -O $HOME/.zetacored/config/addrbook.json  https://server-4.itrocket.net/testnet/zetachain/addrbook.json
```

***set seeds and peers**
```
SEEDS="c1bbbfe2a5b15674bf24a869b3e8189b6b410ae7@zetachain-testnet-seed.itrocket.net:14656"
PEERS="d21b103628b0d5d824bbe81b809d8dc457bd2059@zetachain-testnet-peer.itrocket.net:16656,66338a18a755a0c780b011f012ff142ebaa8fa56@44.236.174.26:26656,e3fea0450f9d23ad7b64d41aab882a82a0b71d6b@150.136.176.81:26656,e3a9810a22a12c04ef1663f8747274e4ef1bdf58@51.159.145.74:26656,c1355344beed2224ff1377dd102e6f847cce2cb6@34.253.137.241:26656,1ef4e7193a9d42a8f3e5195f7a21e925bc50ff55@57.129.28.218:26656,a6090cdf3ff4bdc428ba89c4f622ec1b3490e338@18.143.71.236:26656,2352e5f3bad70d13ebae1876966d6a10c219e819@95.216.244.70:26656,7b1e4c6dbdeb65fbaafd5fe415679d252bdae2f9@141.94.214.137:26656,3dba050e7733ac755386015a0aa7adafe99120c7@135.181.136.250:16656,57693a9bce3ffb5d6023a161ac9f744ac09a2329@162.19.240.28:26656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.zetacored/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${ZETACHAIN_PORT}317%g;
s%:8080%:${ZETACHAIN_PORT}080%g;
s%:9090%:${ZETACHAIN_PORT}090%g;
s%:9091%:${ZETACHAIN_PORT}091%g;
s%:8545%:${ZETACHAIN_PORT}545%g;
s%:8546%:${ZETACHAIN_PORT}546%g;
s%:6065%:${ZETACHAIN_PORT}065%g" $HOME/.zetacored/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${ZETACHAIN_PORT}658%g;
s%:26657%:${ZETACHAIN_PORT}657%g;
s%:6060%:${ZETACHAIN_PORT}060%g;
s%:26656%:${ZETACHAIN_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${ZETACHAIN_PORT}656\"%;
s%:26660%:${ZETACHAIN_PORT}660%g" $HOME/.zetacored/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.zetacored/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.zetacored/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.zetacored/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0azeta"|g' $HOME/.zetacored/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.zetacored/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.zetacored/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/zetacored.service > /dev/null <<EOF
[Unit]
Description=Zetachain node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.zetacored
ExecStart=$(which zetacored) start --home $HOME/.zetacored --log_format json  --log_level info --moniker $MONIKER
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
zetacored tendermint unsafe-reset-all --home $HOME/.zetacored
if curl -s --head curl https://server-4.itrocket.net/testnet/zetachain/zetachain_2024-12-09_8031465_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/zetachain/zetachain_2024-12-09_8031465_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.zetacored
    else
  echo "no snapshot found"
fi
```

**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable zetacored
sudo systemctl restart zetacored && sudo journalctl -u zetacored -f
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/zetachain/autoinstall/)
```

Create wallet
**to create a new wallet, use the following command. don’t forget to save the mnemonic**
```
zetacored keys add $WALLET
```

**to restore exexuting wallet, use the following command**
```
zetacored keys add $WALLET --recover
```

**save wallet and validator address**
```
WALLET_ADDRESS=$(zetacored keys show $WALLET -a)
VALOPER_ADDRESS=$(zetacored keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**check sync status, once your node is fully synced, the output from above will print "false"**
```
zetacored status 2>&1 | jq 
```

# before creating a validator, you need to fund your wallet and check balance
zetacored query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.zetacored/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://zetachain-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

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
Amount, azeta
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
zetacored tx staking create-validator \
--amount 1000000azeta \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(zetacored tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id athens_7001-1 \
--gas auto --gas-adjustment 1.5 --fees 5136570000000000azeta \
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
sudo ufw allow ${ZETACHAIN_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop zetacored
sudo systemctl disable zetacored
sudo rm -rf /etc/systemd/system/zetacored.service
sudo rm $(which zetacored)
sudo rm -rf $HOME/.zetacored
sed -i "/ZETACHAIN_/d" $HOME/.bash_profile
