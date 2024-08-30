# Running a Dusk 10 Sequencer Full Node

> Note: This was tested on a Google Cloud Platform `c3d-standard-4` (4 vCPU,
> 16GB RAM, 500GB SSD) running `debian-12-bookworm`.

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

> Note: make sure to install go 1.22

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
git checkout origin/v0.38.x
export GOPATH=~/go
make install
cd ../
```

### Sequencer

```bash
git clone https://github.com/astriaorg/astria.git
cd astria
git checkout tags/sequencer-v0.16.0
cd crates/astria-sequencer/
cargo build --release
cd ../../../
```

## Configure

### Setup CometBFT `config.toml`

Timeout commit needs to be set to 2s, and peers are those listed in [peers.txt](./peers.txt)

```bash
cometbft init
sed -i'.bak' 's/timeout_commit = "1s"/timeout_commit = "1500ms"/g' ~/.cometbft/config/config.toml
sed -i'.bak' 's/persistent_peers = ""/persistent_peers = "f4b8a8dcfc5a142bd00aadab71f39dbfe7091d13@35.236.107.39:26656,ca3bc3562919b82575fe3ac5b11fa5962ce8cd3b@34.118.249.169:26656,4418000e355967ecc8e03004f5850dfde51c410b@34.94.177.254:26656,7a117e7906d8428ad20341aca94af03c980c11d8@34.102.55.164:26656"/g' ~/.cometbft/config/config.toml
```

You may also want to enable other settings within the config.toml, such as
enabling prometheus metrics. See [CometBFT
docs](https://docs.cometbft.com/v0.38/core/configuration) for more information
on configuration.

### Get Genesis File

```bash
curl -o genesis.json -s https://raw.githubusercontent.com/astriaorg/networks/main/dusk-10/sequencer/genesis.json
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

The first sequencer block which has a tx is `458` which has hash

```bash
BBF365C8E93CBDDAEB3567157DCA1F8B536AEC4246BB24510CBFAC3792C75DD3
```

### Check primary sequencer node

```bash
curl -X GET "https://rpc.sequencer.dusk-10.devnet.astria.org/block?height=458" \
  -H "accept: application/json" -s \
  | jq .result.block_id.hash 
```

### Check your local node

```bash
curl -X GET "http://localhost:26657/block?height=458" \
  -H "accept: application/json" -s \
  | jq .result.block_id.hash
```

## Via Helm Chart

### Add Astria Helm Charts Repo

```bash
helm repo add astria https://astriaorg.github.io/charts/
```

### Pull values file to use

```bash
curl -o full-node-values.yaml -s https://raw.githubusercontent.com/astriaorg/networks/main/dusk-10/sequencer/full-node-values.yaml
```

Note that you may want to edit this values file, by default it assumes you are
installing somewhere such as a cloud provider that can create PVCs and already
has default storage classes.

### Install

```bash
helm install dusk10-full-node astria/astria-sequencer-validator --version 0.16.0 \
  --namespace astria-dusk10-node --create-namespace -f full-node-values.yaml
