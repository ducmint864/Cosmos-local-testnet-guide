#  Tutorial: Simple cosmos testnet with 2 validator nodes

## 1. Prerequisites
``
Ignite CLI
``

## 2. Initialize your chain

Create a binary in your terminal:
```bash
ignite scaffold chain <application-name>
```
Example:
```bash
ignite scaffold chain testchain
```
Build the application chain:
```bash
cd <application-name>
ignite chain build
```
Cosmos SDK will automatically create an executable binary cli file, and we can use it as a command:
```bash
<application-name>d

* If application-name is testchain, then the command would be:
testchaind

* If the command is not recognized by the system, add go/bin go your path as cosmos put the binary file in /go/bin by default
```

| `
From now on, we will assume that your application-name was testchain. Therefore, the binary cli file should be testchaind
`|
## 3. Setup testnet environment
### 3.1 Create testnet folder and nodes folder
```bash
cd ~
mkdir testnet && cd testnet
mkdir node1; mkdir node2; mkdir node1/node1home; mkdir node2/node2home
```
### 3.2 Create test accounts(keys) for each node
Create test account for node1:
```
testchaind keys add node1-account --home node1/node1home/
```
Create test account for node2:
```bash
testchaind keys add node2-account --home node2/node2home/
```

## 4. Initialize chain nodes
From the ~/testnet folder, run:
```bash
testchaind init node1 --home node1/node1home/
testchaind init node2 --home node2/node2home/
```

## 5. Add genesis accounts (Only run on 1 node)
From the ~/testnet folder, run:
```bash
testchaind genesis add-genesis-account $(testchaind keys show node1-account -a --home node1/node1home) 100000000000000000000000000stake --home node1/node1home
testchaind genesis add-genesis-account $(testchaind keys show node2-account -a --home node2/node2home) 100000000000000000000000000stake --home node1/node1home
```

## 6. Stake into the chain (Only run on 1 node)
From the ~/testnet folder, run:
```bash
testchaind genesis gentx node1-account 10000000000stake --fees 100000stake --home node1/node1home
```
Add the generated gentx transaction to genesis.json:
```bash
testchaind genesis collect-gentxs --home node1/node1home/ --trace
```
## 7. Start the chain node (run single-node)
Before you start, the api server and swagger should be enabled. Go to the **node1/node1home/config/app.toml** to change those options as follow:
/image here

Give a value to min gas price config variable of the chain which is left blank by default. Open the file **node1/node1home/config/app.toml** and set variable **minimum-gas-prices** to your desired value. For example:
```toml
minimum-gas-prices = "0.1stake"
```
Then start the chain:
```bash
testchaind start --home node1/node1home
```

## 8. Config node2 to sync with the blockchain on node1
### 8.1 Copy chain configuration from node1 to node2
First of all, go back to ~/testnet folder:
```bash
cd ~/testnet
```
Copy all the config files from node1 to node2 by running the below command:
- genesis.json
- app.toml
- client.toml
- config.toml

```bash
cp node1/node1home/config/{genesis.json,config.toml,app.toml,client.toml} node2/node2home/config/
```

Change every ports in these 3 .toml files to differ from those of node1
* Open **node2/node2home/config/app.toml**  file and edit ports as follow:
```toml
# Address defines the API server to listen on.
address = "tcp://localhost:1318"
```
```toml
# Address defines the gRPC server address to bind to.
address = "localhost:9092"
```
* Open **node2/node2home/config/config.toml**  file and edit ports as follow:
```toml
# TCP or UNIX socket address of the ABCI application,
# or the name of an ABCI application compiled in with the CometBFT binary
proxy_app = "tcp://127.0.0.1:26659"
```
```toml
# TCP or UNIX socket address for the RPC server to listen on
laddr = "tcp://127.0.0.1:26654"
```
```toml
# pprof listen address (https://golang.org/pkg/net/http/pprof)
pprof_laddr = "localhost:6062"
```
```toml
#######################################################
###           P2P Configuration Options             ###
#######################################################
[p2p]

# Address to listen for incoming connections
laddr = "tcp://0.0.0.0:26653"
```
* Open **node2/node2home/config/client.toml**  file and edit ports as follow:
```toml
# <host>:<port> to CometBFT RPC interface for this chain
node = "tcp://localhost:26654"
```

### 8.2 Add node1 to node2's list of peers
Open **node2/node2home/config/config.toml** file and edit the 'seeds' and 'persistent_peers' options as follow:
```toml
# Comma separated list of seed nodes to connect to
seeds="<node1-ID>@127.0.0.1:26656"

# Comma separated list of nodes to keep persistent connections to
persistent_peers="<node1-ID>@127.0.0.1:26656"
```
``To get your node1-ID, run command: ``
```bash
testchaind comet show-node-id --home node1/node1home/
```
### 8.3 Start node2
Before running node2, you should start node1 (if it hasn't been started yet) or restart (if it's been closed):
```bash
testchaind start --home node1/node1home/
```
Start node2 so that it run alongside of node1:
```bash
testchaind start --home node2/node2home/
```
NOTE: in local development, if using 127.0.0.1 and the node throws an error, you can try `localhost instead.

**At this point, the screen output of the 2 nodes should be in sync**
## 9. Run node2 as a validator (optional)
##### This section is aimed at upgrading the role of node2 from normal participant node to a validator node for our blockchain
### 9.1 Create a validator_info.json file
[WHY?] Because the command we're gonna use to make node2 a validator is ``testchaind tx staking create-validator``. That command requires a .json file which entails information about the new validator to be passed in as command argument. Therefore, we go ahead to create a .json file:
```bash
cd ~/testnet/node2
touch validator_info.json
```
Put this json bundle into validator_info.json file, the file's content would look like this:
```json
{
	"pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"NODE2-VALIDATOR-PUBKEY"},
	"amount": "10000000000stake",
	"moniker": "node2",
	"identity": "",
	"website": "",
	"security": "",
	"details": "",
	"commission-rate": "0.1",
	"commission-max-rate": "0.2",
	"commission-max-change-rate": "0.01",
	"min-self-delegation": "1"
}
```
NOTE: you have to replace ``NODE2-VALIDATOR-PUBKEY`` with the real pubkey of node2's validator account. A convenient way to retrieve such pubkey is to run ``gentx`` command on node2-account and then copy the validator's pubkey from the result file. The steps are as follow:
Run gentx command first:
```bash
testchaind genesis gentx node2-account 10000000000stake --home node2/node2home/

Output>  Genesis transaction written to "node2/node2home/config/gentx/gentx-98eae84d5327de5857952c909eb1ac1ce3d3bd6f.json"
```
Open the result file
```bash
nano node2/node2home/config/gentx/gentx-98eae84d5327de5857952c909eb1ac1ce3d3bd6f.json
```
There should be a pubkey field like this in the result file:
```json
				"pubkey": {
					"@type": "/cosmos.crypto.ed25519.PubKey",
					"key": "g3u70aVnX6B4CaeQuf+zE6qOTEyS12jl5Wnphn/kLLM="
				},
```
Copy the key. In my case it was: ``g3u70aVnX6B4CaeQuf+zE6qOTEyS12jl5Wnphn/kLLM=``

Now re-open the **node2/validator_info.json** file and replace ``NODE2_VALIDATOR-PUBKEY`` with real value above:
```json
{
	"pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"g3u70aVnX6B4CaeQuf+zE6qOTEyS12jl5Wnphn/kLLM="},
	"amount": "10000000000stake",
	"moniker": "node2",
	"identity": "",
	"website": "",
	"security": "",
	"details": "",
	"commission-rate": "0.1",
	"commission-max-rate": "0.2",
	"commission-max-change-rate": "0.01",
	"min-self-delegation": "1"
}
```

### 9.2 Add validator
To add node2 as a validator, run command:
```bash
testchaind tx staking create-validator node2/validator_info.json --from node2-account --home node2/node2home --fees 100000stake --chain-id testchain
```

### 9.3 Verify that node2 has been added to the validator list of the chain
Since there are just 2 possible validators in the chain, you can print out the current set of validators and see if it contains node2's validator. Run the following command:
```bash
testchaind q comet-validator-set --home node1/node1home/
```

If correct, the command should output details about 2 validators similar to this one:
```yaml
block_height: "614"
pagination:
  next_key: null
  total: "2"
validators:
- address: cosmosvalcons1s6ew8n67xxyml2dyw0zqsclry08tulk70kcr06
  proposer_priority: "8750"
  pub_key:
    '@type': /cosmos.crypto.ed25519.PubKey
    key: g3u70aVnX6B4CaeQuf+zE6qOTEyS12jl5Wnphn/kLLM=
  voting_power: "10000"
- address: cosmosvalcons1nvqvtggjz9cy5hd73hslk97709drfe6cm9alnv
  proposer_priority: "-8750"
  pub_key:
    '@type': /cosmos.crypto.ed25519.PubKey
    key: /VRvqELNJhOe+u5ty1nft2INkhLo7lu/7RljxQJPoO0=
  voting_power: "10000"
```
NOTE: Apparently, the pubkey value of the first validator in the output is: ``g3u70aVnX6B4CaeQuf+zE6qOTEyS12jl5Wnphn/kLLM``, we can indeed verify that it matches with node2 validator's pubkey, so node2 has been successfully registered as a validator of the chain.
