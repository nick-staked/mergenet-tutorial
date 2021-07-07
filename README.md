# Mergenet Withdrawals tutorial

It's a local eth1-eth2 merge testnet with **Withdrawals** feature. The last is implemented according to the [Eth2 to Eth1 Validator’s withdrawal design [WIP]](https://hackmd.io/@zilm/withdrawal-spec).   
Let's set up testnet and make it running!

## Preparing the setup environment

In this tutorial, we use a series of scripts to generate configuration
files, and these scripts have dependencies that we need to
install. You can either install these dependencies on your host or you
can run those scripts inside a docker container. We call this
environment setupenv.

Preparing the setup environment on your host:
```shell
apt-get install python3-dev python3-pip python3-venv golang

# Check that you have Go 1.16+ installed
go version

# Create, start and install python venv
python -m venv venv 
. venv/bin/activate
pip install -r requirements.txt

# Install eth2-testnet-genesis tool (Go 1.16+ required)
go install github.com/zilm13/eth2-testnet-genesis@068d0f9eff0b56e6b2b90b9bdad23de9f1447ac5
# Install eth2-val-tools
go install github.com/protolambda/eth2-val-tools@latest
# You are now in the right directory to run the setupenv commands below.
```

Alternatively, you can use docker:
```shell
docker build -t setupenv .
docker run -i -v $PWD:/mergenet-tutorial \
  -v /etc/passwd:/etc/passwd -h setupenv \
  -u $(id -u):$(id -g) -t setupenv \
  /bin/bash
# docker spawns a shell and inside that run:
cd /mergenet-tutorial
# You are now in the right directory to run the setupenv commands below.
```

## Create chain configurations

Set `eth1_genesis_timestamp` inside `mergenet.yaml`to the current
timestamp or a timestamp in the future. To use the current timestamp
run:
```shell
sed -i -e s/GENESIS_TIMESTAMP/$(date +%s)/ mergenet.yaml
```

Otherwise tweak mergenet.yaml as you like. The current default is to
have the Eth2 genesis 10 minutes after the Eth1 genesis.

```shell
# Inside your setupenv: Generate ETH1/ETH2 configs
export TESTNET_NAME="mynetwork"
mkdir -p "$TESTNET_NAME/public" "$TESTNET_NAME/private"
# Configure Eth1 chain
python generate_eth1_conf.py > "$TESTNET_NAME/public/eth1_config.json"
# Configure Eth2 chain
python generate_eth2_conf.py > "$TESTNET_NAME/public/eth2_config.yaml"
```

Configure tranche(s) of validators, edit `genesis_validators.yaml`.
Note: defaults test-purpose mnemonic and configuration is included already, no need to edit for minimal local setup.
Make sure that total of `count` entries is more than the configured `MIN_GENESIS_ACTIVE_VALIDATOR_COUNT` (eth2 config).

## Prepare Eth2 data
```shell
# Inside your setupenv: Generate Genesis Beacon State
eth2-testnet-genesis merge \
  --eth1-config "$TESTNET_NAME/public/eth1_config.json" \
  --eth2-config "$TESTNET_NAME/public/eth2_config.yaml" \
  --mnemonics genesis_validators.yaml \
  --state-output "$TESTNET_NAME/public/genesis.ssz" \
  --tranches-dir "$TESTNET_NAME/private/tranches"

# Build validator keystore for nodes. Different secret is generated per validator keystore.
#
# You can change the range of validator accounts, to split keys between nodes.
# The mnemonic and key-range should match that of a tranche of validators in the beacon-state genesis.
export VALIDATOR_NODE_NAME="valclient0"
eth2-val-tools keystores \
  --out-loc "$TESTNET_NAME/private/$VALIDATOR_NODE_NAME" \
  --prysm-pass="foobar" \
  --source-min=0 \
  --source-max=64 \
  --source-mnemonic="lumber kind orange gold firm achieve tree robust peasant april very word ordinary before treat way ivory jazz cereal debate juice evil flame sadness"
```

## Start nodes

This documents how to build the binaries from source, so you can make changes and check out experimental git branches.
It's possible to build docker images (or use pre-built ones) as well. Ask the client devs for alternative install instructions.

```shell
mkdir clients

mkdir "$TESTNET_NAME/nodes"
```

You can choose to run clients in two ways:
- [Build and run from source](./from_source.md)
- [Run in docker](./from_docker.md)

The docker instructions include how to configure each of the clients. 
Substitute docker volume-mounts with your own directory layout choices, the instructions are otherwise the same. 

## Genesis

Now wait for the genesis of the chain!
`actual_genesis_timestamp = eth1_genesis_timestamp + eth2_genesis_delay`

## Bonus

### Test ETH transaction

Import a pre-mined account into some web3 wallet (e.g. metamask), connect to local RPC, and send a transaction with a GUI.

Run `example_transaction.py`.

### Test withdrawal

You will need several pennies to run withdrawals transaction. There are already 
few pre-mined accounts in generated Eth1 `alloc`, after you add one to Geth 
keystore, you could just unlock it:
```bash
./geth attach ./clients/geth/chaindata/geth.ipc
> web3.personal.unlockAccount("0x5c1fd166fe8d490fdc83d7c32a77711930cc615d", "yourpassword", 10000)
> exit
```
Use following contract ABI for withdrawals system contract:
```javascript
var withdrawalsContract = {
  "abi": [
    {
      "inputs": [],
      "stateMutability": "nonpayable",
      "type": "constructor"
    },
    {
      "anonymous": false,
      "inputs": [
        {
          "indexed": false,
          "internalType": "bytes32",
          "name": "data",
          "type": "bytes32"
        }
      ],
      "name": "Logger",
      "type": "event"
    },
    {
      "inputs": [],
      "name": "deposit",
      "outputs": [],
      "stateMutability": "payable",
      "type": "function"
    },
    {
      "inputs": [
        {
          "internalType": "uint256",
          "name": "slot",
          "type": "uint256"
        }
      ],
      "name": "getRoot",
      "outputs": [
        {
          "internalType": "bytes32",
          "name": "",
          "type": "bytes32"
        }
      ],
      "stateMutability": "nonpayable",
      "type": "function"
    },
    {
      "inputs": [
        {
          "internalType": "uint256",
          "name": "slot",
          "type": "uint256"
        },
        {
          "internalType": "bytes32",
          "name": "expectedRoot",
          "type": "bytes32"
        }
      ],
      "name": "verifyRoot",
      "outputs": [],
      "stateMutability": "nonpayable",
      "type": "function"
    },
    {
      "inputs": [
        {
          "internalType": "uint256",
          "name": "_slot",
          "type": "uint256"
        },
        {
          "internalType": "bytes32[]",
          "name": "_proof",
          "type": "bytes32[]"
        },
        {
          "internalType": "uint64",
          "name": "_gIndex",
          "type": "uint64"
        },
        {
          "components": [
            {
              "internalType": "uint64",
              "name": "validator_index",
              "type": "uint64"
            },
            {
              "internalType": "bytes32",
              "name": "withdrawal_credentials",
              "type": "bytes32"
            },
            {
              "internalType": "uint64",
              "name": "withdrawn_epoch",
              "type": "uint64"
            },
            {
              "internalType": "uint64",
              "name": "amount",
              "type": "uint64"
            }
          ],
          "internalType": "struct Withdrawal",
          "name": "_withdrawal",
          "type": "tuple"
        }
      ],
      "name": "withdraw",
      "outputs": [],
      "stateMutability": "nonpayable",
      "type": "function"
    }
  ]
}
```
Next, you need to initialize **Withdrawals Contract** in **Web3.js** console or using 
a similar tool. Contract address is defined in `mergenet.yaml`.
```javascript
const web3 = new Web3('ws://localhost:8546') // Connect to Geth
var withdrawalsContractInstance = new web3.eth.Contract(withdrawalsContract.abi, "0x4343434343434343434343434343434343434343")
// Check it's good by getting not very old Beacon Block Root (latest 256 are available)
withdrawalsContractInstance.methods.getRoot(500).call().then(console.log)
```
You should get the actual beacon block root as the reply to the latest command. Since 
that we are ready to execute withdrawal!  
First, let's initiate voluntary exit for one of the validators. It's the only way of
withdrawal which is currently supported. Note, that at least `SHARD_COMMITTEE_PERIOD` 
should be passed until you are eligible to exit. In tranches `tranche_0000.txt` you 
could see validators' public keys and order of deposits. Choose one, its `line - 1` 
will be a validator index, remember it, you will need it later. Submit validator exit.
In Teku it will look like this:
```bash
./teku voluntary-exit \
  --validator-keys="$TESTNET_NAME/private/valclient0/teku-keys/0xb8747871053f40070805ad2773328b834613292ce02aa29b030a99dfb114be26a42de8d4f9fef36dc60050c77771991d.json:$TESTNET_NAME/private/valclient0/teku-secrets/0xb8747871053f40070805ad2773328b834613292ce02aa29b030a99dfb114be26a42de8d4f9fef36dc60050c77771991d.txt"
```
Json-file with the name equal to public key contains encrypted validators private key
data and txt-file contains decrypt password. After that validator should pass several 
statuses until withdrawal is eligible:
- Exiting
- Exited unslashed
- Withdrawal possible
- Withdrawal done  

It should take about 21 epochs with default testnet configuration. After that you 
should get `Withdrawal` proof from your Ethereum 2 client. For a client implemented 
JSON-RPC withdrawals API it will look like this (change validator index to your 
actual value):
```bash
> curl -X GET "http://localhost:5051/eth/v1/beacon/states/head/withdrawal?validator_index=35"
{"data":{"beacon_block_root":"0x6df1bde7445f7dae22e293ab2ddfd5e18c9f725b5d059caa9af9c6b375890419","slot":"1402","proof":["0x0000000000000000000000000000000000000000000000000000000000000000","0xf5a5fd42d16a20302798ef6ed309979b43003d2320d9f0e8ea9831a92759fb4b","0xdb56114e00fdd4c1f85c892bf35ac9a89289aaecb1ebd0a96cde606a748b5d71","0xc78009fdf07fc56a11f122370658a353aaa542ed63e44c4bc15ff4cd105ab33c","0x536d98837f2dd165a55d5eeae91485954472d56f246df256bf3cae19352a123c","0x9efde052aa15429fae05bad4d0b1d7c64da64d03d7a1854a588c2cb8430c0d30","0xd88ddfeed400a8755596b21942c1497e114c302e6118290f91e6772976041fa1","0x87eb0ddba57e35f6d286673802a4af5975e22506c7cf4c64bb6be5ee11527f2c","0x26846476fd5fc54a5d43385167c95144f2643f533cc85bb9d16b782f8d7db193","0x506d86582d252405b840018792cad2bf1259f1ef5aa5f887e13cb2f0094f51e1","0xffff0ad7e659772f9534c195c815efc4014ef1e1daed4404c06385d11192e92b","0x6cf04127db05441cd833107a52be852868890e4317e6a02ab47683aa75964220","0xb7d05f875f140027ef5118a2247bbb84ce8f2f0f1123623085daf7960c329f5f","0xdf6af5f5bbdb6be9ef8aa618e4bf8073960867171e29676f8b284dea6a08a85e","0xb58d900f5e182e3c50ef74969ea16c7726c549757cc23523c369587da7293784","0xd49a7502ffcfb0340b1d7885688500ca308161a7f96b62df9d083b71fcc8f2bb","0x8fe6b1689256c0d385f42f5bbe2027a22c1996e110ba97c171d3e5948de92beb","0x8d0d63c39ebade8509e0ae3c9c3876fb5fa112be18f905ecacfecb92057603ab","0x95eec8b2e541cad4e91de38385f2e046619f54496c2382cb6cacd5b98c26f5a4","0xf893e908917775b62bff23294dbbe3a1cd8e6cc1c35b4801887b646a6f81f17f","0xcddba7b592e3133393c16194fac7431abf2f5485ed711db282183c819e08ebaa","0x8a8d7fe3af8caa085a7639a832001457dfb9128a8061142ad0335629ff23ff9c","0xfeb3c337d7a51a6fbf00b9e34c52e1c9195c969bd4e7a0bfd51d5c5bed9c1167","0xe71f0aa83cc32edfbefa9f4d3e0174ca85182eec9f3a09f6a6c0df6377a510d7","0x31206fa80a50bb6abe29085058f16212212a60eec8f049fecb92d8c8e0a84bc0","0x21352bfecbeddde993839f614c3dac0a3ee37543f9b412b16199dc158e23b544","0x619e312724bb6d7c3153ed9de791d764a366b389af13c58bf8a8d90481a46765","0x7cdd2986268250628d0c10e385c58c6191e6fbe05191bcc04f133f2cea72c1c4","0x848930bd7ba8cac54661072113fb278869e07bb8587f91392933374d017bcbe1","0x8869ff2c22b28cc10510d9853292803328be4fb0e80495e8bb8d271f5b889636","0xb5fe28e79f1b850f8658246ce9b6a1e7b49fc06db7143e8fe0b4f2b0c5523a5c","0x985e929f70af28d0bdd1a90a808f977f597c7c778c489e98d3bd8910d31ac0f7","0xc6f67e02e6e4e1bdefb994c6098953f34636ba2b6ca20a4721d2b26a886722ff","0x1c9a7e5ff1cf48b4ad1582d3f4e4a1004f3b20d8c5a2b71387a4254ad933ebc5","0x2f075ae229646b6f6aed19a5e372cf295081401eb893ff599b3f9acc0c0d3e7d","0x328921deb59612076801e8cd61592107b5c67c79b846595cc6320c395b46362c","0xbfb909fdb236ad2411b4e4883810a074b840464689986c3f8a8091827e17c327","0x55d8fb3687ba3ba49f342c77f5a1f89bec83d811446e1a467139213d640b6a74","0xf7210d4f8e7e1039790e7bf4efa207555a10a6db1dd4b95da313aaa88b88fe76","0xad21b516cbc645ffe34ab5de1c8aef8cd4e7f8d2b51e8e1456adc7563cda206f","0x0100000000000000000000000000000000000000000000000000000000000000","0x0000000000000000000000000000000000000000000000000000000000000000","0x0bfe4e540ebdfbd0604b83b6519c4a1d9dc82148276ad68e5cc2f626dbf70d99","0xf5b97bc6d16b41518089440cace65baf3e4534cbf6b5c1c13a4c01d43f6919df","0xc78009fdf07fc56a11f122370658a353aaa542ed63e44c4bc15ff4cd105ab33c","0x848363af0d2d8046917e6c3ed60b20b5305bf74b9a4c91e25a2da1ceec9d3465","0xbfdda2d39b400e0f8bcade81870bd1ae359585db8a9c6902322a13b3c9061d84","0x50ac997fc5f45d46da854455b2ab6ddf7027527b90cca743c3e68dc207d700e6","0x47167dc6345bf32e9d69c844dbbdbf32bb889808bf3fb946df5b2f9710bcfe9c"],"index":"822434697576448","withdrawal":{"validator_index":"35","withdrawal_credentials":"0x0100000000000000000000003e9eb50199be0d1f480b2856c13d124f2f9c350b","withdrawn_epoch":"152","amount":"32194163618"}}}
```
Let's explain in a few words, what we see there and how it will work:
- `withdrawal` is entity containing all required amount about withdrawal: amount, 
  withdrawal credentials etc
- `slot` is slot for which `proof` is correct and will help to build a tree up to 
  `beacon_block_root`
- `proof` are neighbors of your path from the `withdrawal` element up to beacon 
  block root
- `index` is a generalized index of exact withdrawal element. Every element in 
  BeaconState has its generalized index
  For more information check [Merkle proof formats](https://github.com/ethereum/eth2.0-specs/blob/dev/ssz/merkle-proofs.md)  
- Every BeaconBlock points to some snapshot of BeaconState, which contains withdrawals
- You are going to provide all this info to the **Withdrawals Contract**. 
  Contract knows nothing about any withdrawal or any other element from BeaconState, 
  but it could verify whether Beacon Block root was one as provided for any of the 
  latest 256 beacon blocks. It calculates `withdrawal` element root and using `proof` 
  and `index` builds the tree all the way up to beacon block root. Next, it compares 
  calculated beacon block root with real beacon block root for the provided `slot`. 
  If the roots match, it guarantees that for this `slot` this `withdrawal` was the 
  part of `BeaconState` and fires up the green light for your funds claim

So let's finish it. In `Web3js` console. We already initialized Withdrawals Contract instance (see instructions above) and need to make withdrawal:
```javascript
// Check your withdrawal credentials balance
web3.eth.getBalance('0x3e9eb50199be0d1f480b2856c13d124f2f9c350b').then(web3.utils.fromWei).then(console.log)
0
// Withdraw
withdrawalsContractInstance.methods.withdraw(2322, ["0x0000000000000000000000000000000000000000000000000000000000000000","0xf5a5fd42d16a20302798ef6ed309979b43003d2320d9f0e8ea9831a92759fb4b","0xdb56114e00fdd4c1f85c892bf35ac9a89289aaecb1ebd0a96cde606a748b5d71","0xc78009fdf07fc56a11f122370658a353aaa542ed63e44c4bc15ff4cd105ab33c","0x536d98837f2dd165a55d5eeae91485954472d56f246df256bf3cae19352a123c","0x9efde052aa15429fae05bad4d0b1d7c64da64d03d7a1854a588c2cb8430c0d30","0xd88ddfeed400a8755596b21942c1497e114c302e6118290f91e6772976041fa1","0x87eb0ddba57e35f6d286673802a4af5975e22506c7cf4c64bb6be5ee11527f2c","0x26846476fd5fc54a5d43385167c95144f2643f533cc85bb9d16b782f8d7db193","0x506d86582d252405b840018792cad2bf1259f1ef5aa5f887e13cb2f0094f51e1","0xffff0ad7e659772f9534c195c815efc4014ef1e1daed4404c06385d11192e92b","0x6cf04127db05441cd833107a52be852868890e4317e6a02ab47683aa75964220","0xb7d05f875f140027ef5118a2247bbb84ce8f2f0f1123623085daf7960c329f5f","0xdf6af5f5bbdb6be9ef8aa618e4bf8073960867171e29676f8b284dea6a08a85e","0xb58d900f5e182e3c50ef74969ea16c7726c549757cc23523c369587da7293784","0xd49a7502ffcfb0340b1d7885688500ca308161a7f96b62df9d083b71fcc8f2bb","0x8fe6b1689256c0d385f42f5bbe2027a22c1996e110ba97c171d3e5948de92beb","0x8d0d63c39ebade8509e0ae3c9c3876fb5fa112be18f905ecacfecb92057603ab","0x95eec8b2e541cad4e91de38385f2e046619f54496c2382cb6cacd5b98c26f5a4","0xf893e908917775b62bff23294dbbe3a1cd8e6cc1c35b4801887b646a6f81f17f","0xcddba7b592e3133393c16194fac7431abf2f5485ed711db282183c819e08ebaa","0x8a8d7fe3af8caa085a7639a832001457dfb9128a8061142ad0335629ff23ff9c","0xfeb3c337d7a51a6fbf00b9e34c52e1c9195c969bd4e7a0bfd51d5c5bed9c1167","0xe71f0aa83cc32edfbefa9f4d3e0174ca85182eec9f3a09f6a6c0df6377a510d7","0x31206fa80a50bb6abe29085058f16212212a60eec8f049fecb92d8c8e0a84bc0","0x21352bfecbeddde993839f614c3dac0a3ee37543f9b412b16199dc158e23b544","0x619e312724bb6d7c3153ed9de791d764a366b389af13c58bf8a8d90481a46765","0x7cdd2986268250628d0c10e385c58c6191e6fbe05191bcc04f133f2cea72c1c4","0x848930bd7ba8cac54661072113fb278869e07bb8587f91392933374d017bcbe1","0x8869ff2c22b28cc10510d9853292803328be4fb0e80495e8bb8d271f5b889636","0xb5fe28e79f1b850f8658246ce9b6a1e7b49fc06db7143e8fe0b4f2b0c5523a5c","0x985e929f70af28d0bdd1a90a808f977f597c7c778c489e98d3bd8910d31ac0f7","0xc6f67e02e6e4e1bdefb994c6098953f34636ba2b6ca20a4721d2b26a886722ff","0x1c9a7e5ff1cf48b4ad1582d3f4e4a1004f3b20d8c5a2b71387a4254ad933ebc5","0x2f075ae229646b6f6aed19a5e372cf295081401eb893ff599b3f9acc0c0d3e7d","0x328921deb59612076801e8cd61592107b5c67c79b846595cc6320c395b46362c","0xbfb909fdb236ad2411b4e4883810a074b840464689986c3f8a8091827e17c327","0x55d8fb3687ba3ba49f342c77f5a1f89bec83d811446e1a467139213d640b6a74","0xf7210d4f8e7e1039790e7bf4efa207555a10a6db1dd4b95da313aaa88b88fe76","0xad21b516cbc645ffe34ab5de1c8aef8cd4e7f8d2b51e8e1456adc7563cda206f","0x0100000000000000000000000000000000000000000000000000000000000000","0x0000000000000000000000000000000000000000000000000000000000000000","0xc49febc5e9c189c1766e70c666dd15e5e9b9650a81ef986683d7935b7d33c776","0xbffaf379372f012d206e46e7808ce71a07737aad107c7d5486fe5d2905726ef7","0xc78009fdf07fc56a11f122370658a353aaa542ed63e44c4bc15ff4cd105ab33c","0x3e55726321a3afb9a5777176f0636d0886643a86710d385cada53336d98ae17a","0xd288fd15b8404732572e547bff5dcaa798699f59f5f05268b13dbfaeef1900bd","0xea453dc1398e730d2a59203f39a09e09b10fa0269e61c4e211408e24618fe9e4","0x00cb7f8216f7293626dc0ce018a5f2eb4d63f335051e8816884c5ca71044bc34"], "822434697576448", {"validator_index":"35","withdrawal_credentials":"0x0100000000000000000000003e9eb50199be0d1f480b2856c13d124f2f9c350b","withdrawn_epoch":"289","amount":"32392681781"}).send({from: '0x5C1fd166FE8D490fdc83D7C32a77711930cc615d', gas: 3141592}).then(console.log);
// Check your withdrawal credentials balance
web3.eth.getBalance('0x3e9eb50199be0d1f480b2856c13d124f2f9c350b').then(web3.utils.fromWei).then(console.log)
32.392681781
```
Finally, we get it! 
withdraw method inputs looks like this:
```javascript
withdrawalsContractInstance.methods.withdraw(<slot>, <proof>, <index>, <withdrawal>)
```
Please, note, withdrawals contract is prefunded and withdrawals are not minted yet, 
this feature is considered to be next testnet upgrade. If you want to change prefunded 
amount, you could modify it in `generate_eth1_conf.py`

