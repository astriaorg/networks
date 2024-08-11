# Astria Dusknet 9 Sequencer

## Software Versions

|  Software  | Version | Links |
|------------|---------|-------|
| Sequencer  | v0.15.0  | [Docker](http://ghcr.io/astriaorg/sequencer:0.15.0--sequencer), [Code](https://github.com/astriaorg/astria/tree/sequencer-v0.15.0/crates/astria-sequencer) |
| Sequencer-relayer | v0.16.0 | [Docker] (http://ghcr.io/astriaorg/sequencer-relayer:0.16.0--sequencer-relayer), [Code](https://github.com/astriaorg/astria/tree/sequencer-relayer-v0.16.0/crates/astria-sequencer-relayer) |
| Composer | v0.8.1 | [Docker] (http://ghcr.io/astriaorg/composer:0.8.1--composer), [Code](https://github.com/astriaorg/astria/tree/composer-v0.8.1/crates/astria-composer) |
| Conductor | v0.19.0 | [Docker] (http://ghcr.io/astriaorg/conductor:0.19.0--conductor), [Code](https://github.com/astriaorg/astria/tree/conductor-v0.19.0/crates/astria-conductor) |
| CometBFT   | v0.38.x | [Docker](http://docker.io/cometbft/cometbft:v0.38.x), [Code](https://github.com/cometbft/cometbft/tree/v0.38.x) |


## Config

The `timeout_commit` property for CometBFT should be set to 1500ms
