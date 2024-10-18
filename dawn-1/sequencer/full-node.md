# Running a Dawn 0 Sequencer Full Node

> Note: This was tested on a Google Cloud Platform `c3d-standard-4` (4 vCPU,
> 16GB RAM, 1000GB SSD) running `debian-12-bookworm`.
>
## System Recommendations

Minimum system requirements are a work in progress our recommended
setup:

- 4 vCPU
- 8 GB RAM
- 1TB SSD

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
wget https://go.dev/dl/go1.22.6.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.6.linux-amd64.tar.gz
export GOPATH=~/go
export PATH=$PATH:/usr/local/go/bin
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
git checkout tags/v0.38.11
make install
cd ../
```

### Sequencer

> Note: it is normal for librocksdb to take a while during this build process.

```bash
git clone https://github.com/astriaorg/astria.git
cd astria
git checkout tags/sequencer-v1.0.0-rc.1
cd crates/astria-sequencer/
cargo build --release
cd ../../../
```

## Configure

### Setup CometBFT `config.toml`

Timeout commit needs to be set to 1500ms, and peers are those listed in [peers.txt](./peers.txt)

> ❗ Note the below sets up cometbft directory ~/.dawn-1-cometbft replace all
> instances in the rest of this doc with your preferred cometbft config directory.
> This directory was chosen to avoid conflicts with other cometbft configurations.

```bash
cometbft init --home ~/.dawn-1-cometbft
sed -i'.bak' 's/timeout_commit = "1s"/timeout_commit = "1500ms"/g' ~/.dawn-1-cometbft/config/config.toml
sed -i'.bak' 's/persistent_peers = ""/persistent_peers = "fa453f8fa2e916025e5718bc53af97e9089058a8@35.235.126.231:26656,48d4eb0607a7b27f1af0288e60ac5a4ee6823adc@34.94.126.233:26656,0aaf5845fa6733b519c73644e17e6afcc53bd4a0@35.236.112.124:26656,59bc2e293f36900f403c0bfc5d00cd016c5397c3@34.94.60.57:26656"/g' ~/.dawn-1-cometbft/config/config.toml
```

You may also want to enable other settings within the config.toml, such as
enabling prometheus metrics. See [CometBFT
docs](https://docs.cometbft.com/v0.38/core/configuration) for more information
on configuration.

### Get Genesis File

```bash
curl -o genesis.json -s https://raw.githubusercontent.com/astriaorg/networks/main/dawn-1/sequencer/genesis.json
mv genesis.json ~/.dawn-1-cometbft/config/genesis.json
```

### Setup Validator Key

The cometbft init command creates a validator key file. If you are a part of the
validator set you must replace this key. There are a couple ways to do this.

- replace the file at `~/.dawn-1-cometbft/config/priv_validator_key.json`
with your preffered cometbft validator key file.
- update the config file at `~/.dawn-1-cometbft/config/config.toml`:
  - replace the path in `priv_validator_key_file` with path to your validator key
  - run [tmkms](https://github.com/iqlusioninc/tmkms) seperately
    and configure `priv_validator_laddr` to the signing server location

See documentation in cometbft docs for information about recommended validator
architecture setups [here](https://docs.cometbft.com/v0.38/core/validators#validator-node-configuration).

## Setup & Run using systemd

### Setup Astria Sequencer Env Vars

> ❗ Replace  `/home/astria_org` **everywhere below** with the absolute path to
> your preferred directory. `systemd` requires this to be an absolute path!

#### From `/home/astria_org`

> The .env file copied below has other settipings you may want to change wrt
> general configuration, especially around metrics and logging. It is self
> self documented.

```bash
mkdir /home/astria-org/sequencer-1.0.0-rc.1
cp ~/astria/target/release/astria-sequencer /home/astria-org/sequencer-1.0.0-rc.1/astria-sequencer
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
ExecStart=/home/astria_org/sequencer-1.0.0-rc.1/astria-sequencer

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
ExecStart=/home/astria_org/go/bin/cometbft start --home /home/astria_org/.dawn-1-cometbft

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

The first sequencer block which has a tx is `1052` which has hash

```bash
9DA0330AEDD7148E7DA6A4F760EE16E5C431B19C8CA99D2E2E942A3C6B076366
```

### Check primary sequencer node

```bash
curl -X GET "https://rpc.sequencer.dawn-1.astria.org/block?height=1052" \
  -H "accept: application/json" -s \
  | jq .result.block_id.hash 
```

### Check your local node

```bash
curl -X GET "http://localhost:26657/block?height=1052" \
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
curl -o full-node-values.yaml -s https://raw.githubusercontent.com/astriaorg/networks/main/dawn-1/sequencer/full-node-values.yaml
```

Note that you may want to edit this values file, by default it assumes you are
installing somewhere such as a cloud provider that can create PVCs and already
has default storage classes.

### Install

```bash
helm install dawn-1-full-node astria/sequencer --version 1.0.0-rc.1 \
  --namespace astria-dawn-1-node --create-namespace -f full-node-values.yaml
