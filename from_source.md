# Building clients from source

When making changes to clients for interop purposes, you may not want to use docker.
This doc describes how to build every client from source. There are currently no options, 
only Geth for execution layer and Teku for consensus layer.

This guide assumes a Linux or MacOS install.
For Windows, please refer to the build instructions by the client (if compatible with windows at all).

For instructions how to use the CLI arguments of the nodes, see the [Docker tutorial](./from_docker.md). 

## Execution clients

## Geth

### Prerequisites

1. Install `go`
2. Add Go to your Path:
```shell
export GOPATH=$HOME/go
export GOROOT=/usr/lib/go
export PATH="$PATH:$GOPATH/bin"
```

### Build

```shell
git clone git@github.com:zilm13/go-ethereum.git
cd go-ethereum
git checkout merge-feature/withdrawals

# To build a binary:
go build -o ./build/bin/catalyst ./cmd/geth
# Or, to build a binary and add it to your go bin path:
go install ./cmd/geth
```

### Initialize

To run a custom testnet, Geth needs to initialize its storage and configuration.
Run the `geth --catalyst init` command to initialize.
See [docker instructions](./from_docker.md#initialization) for details.
```shell
./geth \
  --catalyst \
  --datadir "${PWD}/clients/geth/chaindata" \
  init "${PWD}/$TESTNET_NAME/public/eth1_config.json" 
```

### Run

```shell
./geth \
  --catalyst \
  --http --http.api net,eth,consensus \
  --http.port 8545 \
  --http.addr 0.0.0.0 \
  --http.corsdomain "*" \
  --ws --ws.api net,eth,consensus \
  --ws.port 8546 \
  --ws.addr 0.0.0.0 \
  --nodiscover \
  --allow-insecure-unlock \
  --miner.etherbase 0x1000000000000000000000000000000000000000 \
  --datadir "${PWD}/clients/geth/chaindata"
```

----
## Consensus clients

## Teku (TXRX fork)

### Prerequisites

1. Install open-jdk 15+ (or oracle jdk)

### Build

```shell
git clone git@github.com:zilm13/teku.git
cd teku
git checkout withdrawal-simulation

./gradlew distTar installDist -x test
```

### Run
Run a network node
```shell
./build/install/teku/bin/teku --logging debug \
  --network "${PWD}/$TESTNET_NAME/public/eth2_config.yaml" \
  --data-path "${PWD}/clients/teku" \
  --p2p-enabled=true \
  --initial-state "${PWD}/$TESTNET_NAME/public/genesis.ssz" \
  --eth1-endpoint "http://localhost:8545" \
  --p2p-discovery-bootnodes "COMMA_SEPARATED_ENRS_HERE" \
  --metrics-enabled=true --metrics-interface=0.0.0.0 --metrics-port="8000" \
  --p2p-discovery-enabled=true \
  --p2p-peer-lower-bound=1 \
  --p2p-port="9000" \
  --rest-api-port="4000" \
  --metrics-host-allowlist="*" \
  --rest-api-host-allowlist="*" \
  --Xdata-storage-non-canonical-blocks-enabled=true
  # optional:
  # --p2p-advertised-ip=1.2.3.4
```
Run a network validators
```shell
./build/install/teku/bin/teku \
  --network "${PWD}/$TESTNET_NAME/public/eth2_config.yaml" \
  --data-path "${PWD}/clients/teku" \
  --beacon-node-api-endpoint "http://localhost:4000" \
  --validators-graffiti="hello" \
  --validator-keys="${PWD}/$TESTNET_NAME/private/valclient0/teku-keys:${PWD}/$TESTNET_NAME/private/valclient0/teku-secrets"
```
Or locally run a node with validators:
```shell
./build/install/teku/bin/teku --logging debug \
  --network "${PWD}/$TESTNET_NAME/public/eth2_config.yaml" \
  --data-path "${PWD}/clients/teku" \
  --p2p-enabled=false \
  --initial-state "${PWD}/$TESTNET_NAME/public/genesis.ssz" \
  --eth1-endpoint "http://localhost:8545" \
  --p2p-discovery-enabled=false \
  --p2p-port="9000" \
  --rest-api-enabled=true \
  --rest-api-docs-enabled=true \
  --rest-api-interface=0.0.0.0 \
  --validator-keys="${PWD}/$TESTNET_NAME/private/valclient0/teku-keys:${PWD}/$TESTNET_NAME/private/valclient0/teku-secrets"
```

----
