# Astria Dawn 0 Sequencer

## Software Versions

|  Software  | Version | Links |
|------------|---------|-------|
| Sequencer  | v0.16.0  | [Docker](http://ghcr.io/astriaorg/sequencer:0.16.0--sequencer), [Code](https://github.com/astriaorg/astria/tree/sequencer-v0.16.0/crates/astria-sequencer) |
| CometBFT   | v0.38.x | [Docker](http://docker.io/cometbft/cometbft:v0.38.x), [Code](https://github.com/cometbft/cometbft/tree/v0.38.x) |
| Sequencer-relayer | v0.16.1 | [Docker](http://ghcr.io/astriaorg/sequencer-relayer:0.16.1--sequencer-relayer), [Code](https://github.com/astriaorg/astria/tree/sequencer-relayer-v0.16.1/crates/astria-sequencer-relayer) |

## Config

The `timeout_commit` property for CometBFT should be set to 1500ms
