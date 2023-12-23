# Fresh VM

This was tested on a Google Cloud Platform `c3d-standard-4` (4 vCPU, 16GB RAM, 500GB SSD) running `debian-12-bookworm`

# Dependencies

```bash
sudo apt update
```

```bash
sudo apt install git -y
```

```bash
sudo apt install build-essential -y
```

```bash
sudo apt-get install libclang-dev
```

```bash
sudo apt install curl -y
```

```bash
sudo apt install wget -y
```

```bash
sudo apt-get install jq -y
```

```bash
sudo apt install golang -y
export GOPATH=~/go
export PATH=$PATH:$GOPATH/bin
```

```bash
# First, install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Then, source the cargo environment
source $HOME/.cargo/env
```

```bash
cargo install just
```

## CometBFT

```bash
git clone https://github.com/cometbft/cometbft
cd cometbft
git checkout origin/v0.37.x
export GOPATH=~/go
make install
```

## Sequencer

```bash
git clone https://github.com/astriaorg/astria.git
cd astria
git checkout sequencer-v0.7.x
cd crates/astria-sequencer/
cargo build --release
```

## Get Genesis File

```bash
cd ../../..
curl -o genesis.json -s https://raw.githubusercontent.com/astriaorg/networks/main/dusk-2/sequencer/genesis.json
mv genesis.json ~/.cometbft/config/genesis.json
```

## Setup CometBFT `config.toml`

```bash
cometbft init
sed -i'.bak' 's/timeout_commit = "1s"/timeout_commit = "2s"/g' ~/.cometbft/config/config.toml
sed -i'.bak' 's/persistent_peers = ""/persistent_peers = "0b92c85f6d74cbc7fb72d91b14d0dfd504151088@35.236.31.7:26656,8dc97ea4864d163456aec5c7b4a85236a9d96a1c@34.94.84.182:26656,7ff274fea878b88575e2c85a2c7d748bfbc9eb4c@34.94.9.152:26656"/g' ~/.cometbft/config/config.toml
```

> ❗ Replace  `/home/josh_astria_org` **everywhere below** with the absolute path to your home directory. `systemd` requires this to be an absolute path!

## Setup Astria Sequencer Env Vars

### From `~`

```bash
cp ~/astria/crates/astria-sequencer/local.env.example astria-sequencer.env
mkdir astria_db
sed -i'.bak' 's/"\/tmp\/astria_db"/"\/home\/josh_astria_org\/astria_db"/g' ~/astria-sequencer.env

```

## astria-sequencer.service

```bash
cat << 'EOF' > astria-sequencer.service
[Unit]
Description=astria sequencer

[Service]
EnvironmentFile=/home/josh_astria_org/astria-sequencer.env
ExecStart=/home/josh_astria_org/astria/target/release/astria-sequencer

# Restart service on failure
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

## Install and run in systemd

```bash
sudo mv astria-sequencer.service /etc/systemd/system/astria-sequencer.service
sudo systemctl daemon-reload
sudo systemctl enable astria-sequencer.service
sudo systemctl start astria-sequencer.service
```

### Check Status

```bash
sudo systemctl status astria-sequencer.service
```

### Check Logs

```bash
sudo journalctl -u astria-sequencer.service
```

## cometbft.service

```bash
cat << 'EOF' > cometbft.service
[Unit]
Description=cometbft

[Service]
ExecStart=/home/josh_astria_org/go/bin/cometbft start --home /home/josh_astria_org/.cometbft

# Restart service on failure
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### Install and run in systemd

```bash
sudo mv cometbft.service /etc/systemd/system/cometbft.service
sudo systemctl daemon-reload
sudo systemctl enable cometbft.service
sudo systemctl start cometbft.service
```

### Check Status

```bash
sudo systemctl status cometbft.service
```

### Check Logs

```bash
sudo journalctl -u cometbft.service
```

# Checking for matching block hashes

The first sequencer block which has a tx is `7315` which has hash

```bash
2E3DABC2CDB081D5DFE0D7E135F44307F7593B1F90D7F2D8BAFD336EB284DBD7
```

### Check primary sequencer node

```bash
curl -X GET "https://rpc.sequencer.dusk-2.devnet.astria.org/block?height=7316" -H "accept: application/json" -s | jq .result.block.header.last_block_id.hash
```

### Check your local node

```bash
curl -X GET "http://localhost:26657/block?height=7316" -H "accept: application/json" -s | jq .result.block.header.last_block_id.hash
```