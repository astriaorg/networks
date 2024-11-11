# Running a Astria Mainnet Alpha Geth Full Node

> Note: This was tested on a Google Cloud Platform `c3d-standard-4` (4 vCPU,
> 16GB RAM, 1000GB SSD) running `debian-12-bookworm`.

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

### Astria-geth

```bash
git https://github.com/astriaorg/astria-geth
cd astria-geth
git checkout tags/v1.0.0
make geth
cd ../
```

### conductor

> Note: it is normal for librocksdb to take a while during this build process.

```bash
git clone https://github.com/astriaorg/astria.git
cd astria
git checkout tags/conductor-v1.0.0
cd crates/astria-conductor/
cargo build --release
cd ../../../
```

## Configure

### Setup geth

> ❗ Note the below sets up cometbft directory ~/home/astria-geth replace all
> instances in the rest of this doc with your preferred cometbft config directory.
> This directory was chosen to avoid conflicts with other cometbft configurations.

Get Genesis and Env Var Files

```bash
curl -o genesis.json -s https://raw.githubusercontent.com/astriaorg/networks/refs/heads/main/flame/genesis.json
curl -o genesis.json -s https://raw.githubusercontent.com/astriaorg/networks/refs/heads/main/flame/astria-geth-env-example
mv genesis.json ~/home/astria-geth/genesis.json
mv astria-geth-env-example ~/home/astria-geth/astria-geth-env-example

```

configure geth

```bash
./build/bin/geth init --datadir ~/home/astria-geth/flame --db.engine="pebble" ~/home/astria-geth/genesis.json
```

## Setup & Run using systemd

### Setup Astria Geth Env Vars

> ❗ Replace `/home/astria_org` **everywhere below** with the absolute path to
> your preferred directory. `systemd` requires this to be an absolute path!

#### From `/home/astria_org`

> The .env file copied below has other settipings you may want to change wrt
> general configuration, especially around metrics and logging. It is self
> self documented.

```bash
mkdir /home/astria_org/astria-geth-v1.0.0
cp ~/astria-geth/build/bin/geth /home/astria_org/astria-geth-v1.0.0/geth
cp ~/astria-geth/astria-geth-env-example /home/astria_org/astria-geth.env
```

### astria-geth.service

```bash
cat << 'EOF' > astria-geth.service
[Unit]
Description=astria geth

[Service]
EnvironmentFile=/home/astria_org/astria-geth.env
ExecStart=/home/astria_org/astria-geth-v1.0.0/geth

# Restart service on failure
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### Install and run Geth in systemd

```bash
sudo mv astria-geth.service /etc/systemd/system/astria-geth.service
sudo systemctl daemon-reload
sudo systemctl enable astria-geth.service
sudo systemctl start astria-geth.service
```

#### Check Geth Status

```bash
sudo systemctl status astria-geth.service
```

#### Check Geth Logs

```bash
sudo journalctl -u astria-geth.service
```

### Setup Astria Conductor Env Vars

> ❗ Replace `/home/astria_org` **everywhere below** with the absolute path to
> your preferred directory. `systemd` requires this to be an absolute path!

#### From `/home/astria_org`

> The .env file copied below has other settipings you may want to change wrt
> general configuration, especially around metrics and logging. It is self
> self documented.

```bash
mkdir /home/astria_org/conductor-v1.0.0
cp ~/astria/target/release/astria-conductor /home/astria_org/conductor-v1.0.0/astria-conductor
cp ~/astria/crates/astria-conductor/local.env.example /home/astria_org/astria-conductor.env
```

### astria-conductor.service

```bash
cat << 'EOF' > astria-conductor.service
[Unit]
Description=astria conductor

[Service]
EnvironmentFile=/home/astria_org/astria-conductor.env
ExecStart=/home/astria_org/conductor-v1.0.0/astria-conductor

# Restart service on failure
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### Install and run Conductor in systemd

```bash
sudo mv astria-conductor.service /etc/systemd/system/astria-conductor.service
sudo systemctl daemon-reload
sudo systemctl enable astria-conductor.service
sudo systemctl start astria-conductor.service
```

#### Check Conductor Status

```bash
sudo systemctl status astria-conductor.service
```

#### Check Conductor Logs

```bash
sudo journalctl -u astria-conductor.service
```

## Via Helm Chart

### Add Astria Helm Charts Repo

```bash
helm repo add astria https://astriaorg.github.io/charts/
```

### Pull values file to use

```bash
curl -o full-node-values.yaml -s https://raw.githubusercontent.com/astriaorg/networks/main/flame/flame-node-values.yaml
```

Note that you may want to edit this values file, by default it assumes you are
installing somewhere such as a cloud provider that can create PVCs and already
has default storage classes.

### Install

```bash
helm install astria-flame-node evm-stack --version 1.0.3 \
  --namespace astria-flame-node --create-namespace -f flame-node-values.yaml
```
