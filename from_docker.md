# Running nodes with Docker

This doc guides you through running each client
with custom testnet configuration for testing the Rayonism merge prototypes.

All of the instructions are very verbose about port configuration,
so you can change the ports easily, and avoid port overlaps. 

Each of the instructions assumes you want to mount 
testnet configuration, a datadir for the node, maybe a genesis-state for beacon nodes, and maybe keys/secrets for validators.

It's recommended you create the data dirs on the host, so docker doesn't have to during the run, and permissions match the user-permissions in the container.
There are currently no options, only Geth for execution layer and Teku for consensus layer..

## Execution clients

### Geth

[Docs](https://geth.ethereum.org/docs/install-and-build/installing-geth#run-inside-docker-container)
```
mkalinin/client-go:withdrawals
```

#### Initialization

Important: for custom testnets, you will need to init the data-dir before running the client, like below:

```shell
mkdir -p ${PWD}/$TESTNET_NAME/nodes/geth0
docker run \
  --name tmpgeth \
  --rm \
  -u $(id -u):$(id -g) --net host \
  -v ${PWD}/$TESTNET_NAME/public/eth1_config.json:/networkdata/eth1_config.json \
  -v ${PWD}/$TESTNET_NAME/nodes/geth0:/gethdata \
  mkalinin/client-go:withdrawals \
  --catalyst \
  --datadir "/gethdata/chaindata" \
  init "/networkdata/eth1_config.json"
```

#### Running:

```shell
docker run \
  --name geth0 \
  -u $(id -u):$(id -g) --net host \
  -v ${PWD}/$TESTNET_NAME/public/eth1_config.json:/networkdata/eth1_config.json \
  -v ${PWD}/$TESTNET_NAME/nodes/geth0:/gethdata \
  mkalinin/client-go:withdrawals \
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
  --datadir "/gethdata/chaindata"
```
---

## Consensus clients

### Teku

Note starts a beacon node by default, use the `validator` subcommand to run the separate validator-client.

[Docs](https://docs.teku.consensys.net/en/latest/HowTo/Get-Started/Installation-Options/Run-Docker-Image/)

Note: the main repo does not support merge/rayonism yet.
Use the below image by Mikhail, working on the TXRX (a Consensys R&D team) fork: 
```
mkalinin/teku:withdrawals
```

#### Running the beacon node:

```shell
mkdir -p ${PWD}/$TESTNET_NAME/nodes/teku0bn
docker run \
  --name teku0bn \
  -u $(id -u):$(id -g) --net host \
  -v ${PWD}/$TESTNET_NAME/nodes/teku0bn:/beacondata \
  -v ${PWD}/$TESTNET_NAME/public/eth2_config.yaml:/networkdata/eth2_config.yaml \
  -v ${PWD}/$TESTNET_NAME/public/genesis.ssz:/networkdata/genesis.ssz \
  mkalinin/teku:withdrawals \
  --network "/networkdata/eth2_config.yaml" \
  --data-path "/beacondata" \
  --p2p-enabled=true \
  --logging=debug \
  --initial-state "/networkdata/genesis.ssz" \
  --eth1-endpoint "http://localhost:8545" \
  --p2p-discovery-bootnodes "COMMA_SEPARATED_ENRS_HERE" \
  --metrics-enabled=true --metrics-interface=0.0.0.0 --metrics-port="8000" \
  --p2p-discovery-enabled=true \
  --p2p-peer-lower-bound=1 \
  --p2p-port="9000" \
  --rest-api-enabled=true \
  --rest-api-docs-enabled=true \
  --rest-api-interface=0.0.0.0 \
  --rest-api-port="4000" \
  --metrics-host-allowlist="*" \
  --rest-api-host-allowlist="*" \
  --Xdata-storage-non-canonical-blocks-enabled=true
  # optional:
  # --p2p-advertised-ip=1.2.3.4
```

#### Running the validator:

```shell
# prepare keys
NODE_PATH="$TESTNET_PATH/nodes/teku0vc"
mkdir -p "$NODE_PATH"
cp -r "$TESTNET_PATH/private/validator0/teku-keys" "$NODE_PATH/keys"
cp -r "$TESTNET_PATH/private/validator0/teku-secrets" "$NODE_PATH/secrets"

docker run \
  --name teku0vc \
  -u $(id -u):$(id -g) --net host \
  -v ${PWD}/$TESTNET_NAME/nodes/teku0vc:/validatordata \
  -v ${PWD}/$TESTNET_NAME/public/eth2_config.yaml:/networkdata/eth2_config.yaml \
  mkalinin/teku:withdrawals vc \
  --network "/networkdata/eth2_config.yaml" \
  --data-path "/validatordata" \
  --beacon-node-api-endpoint "http://localhost:4000" \
  --validators-graffiti="hello" \
  --validator-keys "/validatordata/keys:/validatordata/secrets"
```

#### Local run  

Alternatively you could run Teku locally with validators in a single command
```shell
NODE_PATH="$TESTNET_PATH/nodes/teku0lv"
mkdir -p "$NODE_PATH"
cp -r "$TESTNET_PATH/private/validator0/teku-keys" "$NODE_PATH/keys"
cp -r "$TESTNET_PATH/private/validator0/teku-secrets" "$NODE_PATH/secrets"
mkdir -p ${PWD}/$TESTNET_NAME/nodes/teku0lv
docker run \
  --name teku0bn \
  -u $(id -u):$(id -g) --net host \
  -v ${PWD}/$TESTNET_NAME/nodes/teku0lv:/validatordata \
  -v ${PWD}/$TESTNET_NAME/public/eth2_config.yaml:/networkdata/eth2_config.yaml \
  -v ${PWD}/$TESTNET_NAME/public/genesis.ssz:/networkdata/genesis.ssz \
  mkalinin/teku:withdrawals \
  --network "/networkdata/eth2_config.yaml" \
  --data-path "/validatordata" \
  --p2p-enabled=false \
  --logging=debug \
  --initial-state "/networkdata/genesis.ssz" \
  --eth1-endpoint "http://localhost:8545" \
  --p2p-discovery-enabled=false \
  --p2p-port="9000" \
  --rest-api-enabled=true \
  --rest-api-docs-enabled=true \
  --rest-api-interface=0.0.0.0 \
  --validator-keys "/validatordata/keys:/validatordata/secrets"
```
---