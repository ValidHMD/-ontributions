# Installation
### Setup validator name
`MONIKER="YOUR_MONIKER_GOES_HERE"`

### Install dependencies
```
sudo apt update
sudo apt-get install git curl build-essential make jq gcc snapd chrony lz4 tmux unzip bc -y
```

### INSTALL GO
```
rm -rf $HOME/go
sudo rm -rf /usr/local/go
cd $HOME
curl https://dl.google.com/go/go1.20.5.linux-amd64.tar.gz | sudo tar -C/usr/local -zxvf -
cat <<'EOF' >>$HOME/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source $HOME/.profile
go version
```

### Install Node
```
git clone https://github.com/munblockchain/mun

cd mun

git checkout v1.0.0

make install
```

### Initialize Node
`mund init "$MONIKER" --chain-id=mun-1`

### Download genesis and addrbook
```
curl -Ls https://snapshots.ndnodes.com/snapshots/mun/genesis.json > $HOME/.mun/config/genesis.json
curl -Ls https://snapshots.ndnodes.com/snapshots/mun/addrbook.json > $HOME/.mun/config/addrbook.json
```

### Create Service
```
sudo tee /etc/systemd/system/mund.service > /dev/null <<EOF
[Unit]
Description=mund Daemon
After=network-online.target
[Service]
User=$USER
ExecStart=$(which mund) start
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable mund
```

### Download Snapshot
```
mund tendermint unsafe-reset-all --home $HOME/.mun --keep-addr-book 
curl https://snapshots.ndnodes.com/snapshots/mun/snapshot-mun.ndnodes.lz4 | lz4 -dc - | tar
```

# Snapshot
```
sudo systemctl stop mund

cp $HOME/.mun/data/priv_validator_state.json $HOME/.mun/priv_validator_state.json.backup 

mund tendermint unsafe-reset-all --home $HOME/.mun --keep-addr-book 
curl https://snapshots.ndnodes.com/snapshots/mun/snapshot-mun.ndnodes.lz4 | lz4 -dc - | tar -xf - -C $HOME/.mun

mv $HOME/.mun/priv_validator_state.json.backup $HOME/.mun/data/priv_validator_state.json 

sudo systemctl restart mund
sudo journalctl -u mund -f --no-hostname -o cat
```

# State-sync
```
sudo systemctl stop mund

cp $HOME/.mun/data/priv_validator_state.json $HOME/.mun/priv_validator_state.json.backup
mund tendermint unsafe-reset-all --home $HOME/.mun

peers="51b2349d68ce202631e607f9de19003bfa2bc40c@167.235.14.83:37656"  
SNAP_RPC="https://rpc.mun.ndnodes.com:443"

sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.mun/config/config.toml 

LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height);
BLOCK_HEIGHT=$((LATEST_HEIGHT - 1000));
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash) 

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH && sleep 2

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ;
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ;
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ;
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ;
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.terp/config/config.toml

mv $HOME/.mun/priv_validator_state.json.backup $HOME/.mun/data/priv_validator_state.json

sudo systemctl restart mund && sudo journalctl -u mund -f
```
