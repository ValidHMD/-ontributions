# Installation
### Setup validator name
`MONIKER="YOUR_MONIKER_GOES_HERE"`

### Install dependencies
```
sudo apt update
sudo apt-get install git curl build-essential make jq gcc snapd chrony lz4 tmux unzip bc -y
```
### INSTALL GO
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
cd $HOME

rm -rf nibiru

git clone https://github.com/NibiruChain/nibiru.git

cd nibiru

git checkout v1.0.2

make install
```

### Initialize Node
`nibid init "$MONIKER" --chain-id=cataclysm-1`

### Download genesis and addrbook
```
curl -Ls https://snapshots.ndnodes.com/snapshots/nibiru-mainnet/genesis.json > $HOME/.nibid/config/genesis.json
curl -Ls https://snapshots.ndnodes.com/snapshots/nibiru-mainnet/addrbook.json > $HOME/.nibid/config/addrbook.json
```

### Create Service
```
sudo tee /etc/systemd/system/nibid.service > /dev/null <<EOF
[Unit]
Description=nibid Daemon
After=network-online.target
[Service]
User=$USER
ExecStart=$(which nibid) start
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable nibid
```

### Download Snapshot
nibid tendermint unsafe-reset-all --home $HOME/.nibid --keep-addr-book 
curl https://snapshots.ndnodes.com/snapshots/nibiru-mainnet/snapshot-nibiru-mainnet.ndnodes.lz4 | lz4 -dc - | tar -xf - -C $HOME/.nibid
```

### Start the node
```
sudo systemctl restart nibid
journalctl -u nibid -f
```
