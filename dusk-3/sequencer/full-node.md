# Fresh VM

This was tested on a Google Cloud Platform `c3d-standard-4` (4 vCPU, 16GB RAM, 500GB SSD) running `debian-12-bookworm`

## Dependencies

```bash
sudo apt update
```

### Base Dependencies & Tools

```bash
sudo apt install git -y
sudo apt install build-essential -y
sudo apt-get install libclang-dev
sudo apt install curl -y
sudo apt install wget -y
sudo apt-get install jq -y
```

### Golang

Used for CometBFT

```bash
sudo apt install golang -y
export GOPATH=~/go
export PATH=$PATH:$GOPATH/bin
```

### Rust

```bash
# First, install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Then, source the cargo environment
source $HOME/.cargo/env
```

## Build from Source

### CometBFT

```bash
git clone https://github.com/cometbft/cometbft
cd cometbft
git checkout origin/v0.37.x
export GOPATH=~/go
make install
cd ../
```

### Sequencer

```bash
git clone https://github.com/astriaorg/astria.git
cd astria
git checkout tags/sequencer-v0.8.0
cd crates/astria-sequencer/
cargo build --release
cd ../../../
```

## Configure

### Setup CometBFT `config.toml`

Timeout commit needs to be set to 2s, and peers are those listed in [peers.txt](./peers.txt)

```bash
cometbft init
sed -i'.bak' 's/timeout_commit = "1s"/timeout_commit = "2s"/g' ~/.cometbft/config/config.toml
sed -i'.bak' 's/persistent_peers = ""/persistent_peers = "0b92c85f6d74cbc7fb72d91b14d0dfd504151088@34.94.18.158:26656,8dc97ea4864d163456aec5c7b4a85236a9d96a1c@35.235.84.88:26656,7ff274fea878b88575e2c85a2c7d748bfbc9eb4c@34.94.250.38:26656,4e43cc937c6917ae703229878ad1e73ed737f216@34.94.145.24:26656"/g' ~/.cometbft/config/config.toml
```

### Get Genesis File

```bash
curl -o genesis.json -s https://raw.githubusercontent.com/astriaorg/networks/main/dusk-3/sequencer/genesis.json
mv genesis.json ~/.cometbft/config/genesis.json
```

## Setup & Run using systemd

### Setup Astria Sequencer Env Vars

> ❗ Replace  `/home/astria_org` **everywhere below** with the absolute path to your home directory. `systemd` requires this to be an absolute path!

#### From `/home/astria_org`

```bash
cp ~/astria/crates/astria-sequencer/local.env.example /home/astria_org/astria-sequencer.env
mkdir /home/astria_org/astria_db
sed -i'.bak' 's/"\/tmp\/astria_db"/"\/home\/astria_org\/astria_db"/g' /home/astria_org/astria-sequencer.env
```

### astria-sequencer.service

```bash
cat << 'EOF' > astria-sequencer.service
[Unit]
Description=astria sequencer

[Service]
EnvironmentFile=/home/astria_org/astria-sequencer.env
ExecStart=/home/astria_org/astria/target/release/astria-sequencer

# Restart service on failure
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### cometbft.service

```bash
cat << 'EOF' > cometbft.service
[Unit]
Description=cometbft

[Service]
ExecStart=/home/jordan/go/bin/cometbft start --home /home/jordan/.cometbft

# Restart service on failure
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### Install and run Sequencer in systemd

Note, sequencer app must be running before starting CometBFT

```bash
sudo mv astria-sequencer.service /etc/systemd/system/astria-sequencer.service
sudo systemctl daemon-reload
sudo systemctl enable astria-sequencer.service
sudo systemctl start astria-sequencer.service
```

#### Check Sequencer Status

```bash
sudo systemctl status astria-sequencer.service
```

#### Check Sequencer Logs

```bash
sudo journalctl -u astria-sequencer.service
```

### Install and run CometBFT in systemd

```bash
sudo mv cometbft.service /etc/systemd/system/cometbft.service
sudo systemctl daemon-reload
sudo systemctl enable cometbft.service
sudo systemctl start cometbft.service
```

#### Check CometBFT Status

```bash
sudo systemctl status cometbft.service
```

#### Check CometBFT Logs

```bash
sudo journalctl -u cometbft.service
```

## Validate

### Checking for matching block hashes

The first sequencer block which has a tx is `60` which has hash

```bash
CFC6202A46AF949BA758B638876332BEFD7C9F860783818CA77CF250745ADA1C
```

### Check primary sequencer node

```bash
curl -X GET "https://rpc.sequencer.dusk-3.devnet.astria.org/block?height=60" \
  -H "accept: application/json" -s \
  | jq .result.block_id.hash 
```

### Check your local node

```bash
curl -X GET "http://localhost:26657/block?height=60" \
  -H "accept: application/json" -s \
  | jq .result.block_id.hash
```
