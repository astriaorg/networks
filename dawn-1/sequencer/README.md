# Astria Dawn 1 Sequencer

## Software Versions

|  Software  | Version | Links |
|------------|---------|-------|
| Sequencer  | v3.0.0-rc.2  | [Docker](http://ghcr.io/astriaorg/sequencer:3.0.0-rc.2), [Code](https://github.com/astriaorg/astria/tree/sequencer-v3.0.0-rc.2/crates/astria-sequencer) |
| CometBFT   | v0.38.17 | [Docker](http://docker.io/cometbft/cometbft:v0.38.17), [Code](https://github.com/cometbft/cometbft/tree/v0.38.17) |
| Sequencer-relayer | v1.0.0-rc.1 | [Docker](http://ghcr.io/astriaorg/sequencer-relayer:1.0.0-rc.1), [Code](https://github.com/astriaorg/astria/tree/sequencer-relayer-v1.0.0-rc.1/crates/astria-sequencer-relayer) |

## Config

The `timeout_commit` property for CometBFT should be set to 1500ms
