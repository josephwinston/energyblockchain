# Getting Started

This document shows how to use Docker Images to setup the Go-Ethereum
client (geth) on Linux.

## Installation

The instructions here will deploy a **private** Ethereum network on
Ubuntu.  You will need to already have
[Anaconda installed](https://www.anaconda.com/download/) as well as
[Docker CE](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/).
Some of the commands require root access, so you also will need `sudo`
permission.

### Anaconda and Ethereum (Does not work on Windows)

* Create an Anaconda environment where you can install all the python
related software (See
[issues](https://github.com/ConsenSys/ethjsonrpc/issues) related to
`Python 3` on why `Python 2.x` is required for
[ethjsonrpc](https://github.com/ConsenSys/ethjsonrpc)).  This is done
by running:

```bash
sudo apt-get install jq libssl-dev
conda create --name ethereum python=2 pip jupyter scipy numpy pandas matplotlib
source activate ethereum

```

* Next, add `py-solc`, `ethereum`, and
  [ethjsonrpc](https://github.com/emunsing/ethjsonrpc) to you Python installation:

```bash
pip install py-solc
pip install ethereum
pip install git+https://github.com/emunsing/ethjsonrpc
```
### Go Ethereum in a Docker Container

The strategy taken in this example is to place all the private
blockchain information inside of one directory named in the variable
`PRIVATE_BLOCKCHAIN_HOME`.  The first item thatyou will create is a
configuraion file that describes the starting location for the private
blockchain.  Next, you will add users to the configuration file as
well as starting account balances. Then you will start up the
blockchain and start mining.

* Create the default state (*genesis*) and store it in `genesis.json`.
To present unknown computers from connecting, `nonce` should be a
random value and not the constant `0x42`.

```bash
export PRIVATE_BLOCKCHAIN_HOME=/tmp/private-chain
mkdir -p $PRIVATE_BLOCKCHAIN_HOME
tee $PRIVATE_BLOCKCHAIN_HOME/genesis.json <<-EOF
{
  "config": {
        "chainId": 1337,
        "homesteadBlock": 0,
        "eip155Block": 0,
        "eip158Block": 0
    },
  "alloc"      : {},
  "coinbase"   : "0x0000000000000000000000000000000000000000",
  "difficulty" : "0x20000",
  "extraData"  : "",
  "gasLimit"   : "0x2fefd8",
  "nonce"      : "0x0000000000000042",
  "mixhash"    : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp"  : "0x00"
}
EOF
```

* Create a new account on your private blockchain.  This and the
following examples use `--ipcpath` and `--datadir`.  These two
parameters set explicitly so that you can see what the programs are
using.

```bash
docker run -it \
-v $PRIVATE_BLOCKCHAIN_HOME:/private-data \
-v $PRIVATE_BLOCKCHAIN_HOME:/root \
ethereum/client-go:v1.7.0 \
--ipcpath /root/.ethereum/geth.ipc \
--datadir /root/.ethereum \
account new /private-data/genesis.json
```
* Input the password and write down the address.  The previous command
  will return something that looks like:

```bash
WARN [09-29|16:17:21] No etherbase set and no accounts found as default
Your new account is locked with a password. Please give a password. Do not forget this password.
Passphrase:
Repeat passphrase:
Address: {522db6cabbf1fca8c504406659d576f6b2acfc3b}
```

* Prefund some accounts by passing the previous account address in the
  first balance:

```bash
tmp_genesis_file=`mktemp`
jq '.alloc = { "0x522db6cabbf1fca8c504406659d576f6b2acfc3b": {"balance": "111111111"},
               "0x0000000000000000000000000000000000000002": {"balance": "222222222"}
             }' $PRIVATE_BLOCKCHAIN_HOME/genesis.json > $tmp_genesis_file
mv $tmp_genesis_file $PRIVATE_BLOCKCHAIN_HOME/genesis.json
```
* Initialize the node

```bash
docker run \
-v $PRIVATE_BLOCKCHAIN_HOME:/private-data \
-v $PRIVATE_BLOCKCHAIN_HOME:/root \
ethereum/client-go:v1.7.0 \
--ipcpath /root/.ethereum/geth.ipc \
--datadir /root/.ethereum \
init /private-data/genesis.json
```

* Bring up the node.  Note using `--dev` in `1.6.6` and `1.7.0` has
the error
[database already contains an incompatible genesis block](https://github.com/ethereum/go-ethereum/issues/15182).
This example uses ports:

* 8545 for HTTP based JSON RPC API
* 30303 for P2P protocol running the network


```bash
docker run \
-d \
--name ethereum-node \
--net=host \
-v $PRIVATE_BLOCKCHAIN_HOME:/private-data \
-v $PRIVATE_BLOCKCHAIN_HOME:/root \
-p 127.0.0.1:8545:8545 \
-p 0.0.0.0:30303:30303 \
ethereum/client-go:v1.7.0 \
--ipcpath /root/.ethereum/geth.ipc \
--datadir /root/.ethereum \
--rpc --rpcaddr "127.0.0.1" \
--fast --cache=512
```

### Solidity

Install the JavaScript and the binary version of the compiler.  Be
sure to add `SOLC_BINARY` to your environment otherwise `py-solc` will
produce strange error messages like `OSError: [Errno 2] No such file
or directory`.

```bash
sudo npm install -g solc 
python -m solc.install v0.4.16
export SOLC_BINARY=~/.py-solc/solc-v0.4.16/bin/solc
```

## Running

There is a choice here.

* Since `ipcpath` and `datadir` live outside the docker image, you can
start up another docker image to interact with the CLI.  To use this
you would run:

```bash
docker run \
--interactive --tty \
--net=host \
-v $PRIVATE_BLOCKCHAIN_HOME:/private-data \
-v $PRIVATE_BLOCKCHAIN_HOME:/root \
ethereum/client-go:v1.7.0 \
--ipcpath /root/.ethereum/geth.ipc \
--datadir /root/.ethereum \
attach

```

* Otherwise, attach to the `JavaScript` CLI in the `ethereum-node` image

```bash
docker exec -it ethereum-node \
/usr/local/bin/geth \
--ipcpath /root/.ethereum/geth.ipc \
--datadir /root/.ethereum \
attach
```

* Verify the account using `eth.accounts` in the `JavaScript` command
  line (`>`). You should see the same account address that you added
  to the `genesis.json` file.  In this example it is
  `0xe183a341f2fbaac1541575a04d867792d3dc88ed`.

* Ensure you have funds ```eth.getBalance(eth.accounts[0])```.  This
  should be the same value `111111111` you added for the account.

* Unlock the coinbase indefinitely by using `0` for the timeout.
  Replace the `password` with your coinbase password

```
personal.unlockAccount(eth.accounts[0], 'password', 0)
```

* Start mining `miner.start(1)`.  This will return `null`.  If
desired, call `miner.getHashrate()`.  This should return a number
greater than or equal to zero.  Discussions on possible problems and
how to work through them are here:
>https://ethereum.stackexchange.com/questions/16040/why-did-it-returned-null-after-call-miner-start>
**Keep** this terminal open.

### Validate

* Insure everything was installed:

```python
import sys, os, time
from solc import compile_source, compile_files, link_code
from ethjsonrpc import EthJsonRpc

print("Using environment in "+sys.prefix)
print("Python version "+sys.version)
```

This should show you something like:

```python
Python 2.7.13 |Continuum Analytics, Inc.| (default, Dec 20 2016, 23:09:15)
[GCC 4.4.7 20120313 (Red Hat 4.4.7-1)] on linux2
```
* Connect to Ethereum RPC

```python
# Initiate connection to ethereum node
#   Requires a node running with an RPC connection available at port 8545
c = EthJsonRpc('127.0.0.1', 8545)
c.web3_clientVersion()
print(c)
```

The output from the `print` call should be
`u'Geth/v1.7.0-stable/linux-amd64/go1.9'`

* Build the `Hello World` contract

```python
source = """
pragma solidity ^0.4.2;

contract Example {

    string s="Hello World!";

    function set_s(string new_s) {
        s = new_s;
    }

    function get_s() returns (string) {
        return s;
    }
}"""
```

* Compile contract

```python
# Basic contract compiling process.
#   Requires that the creating account be unlocked.
#   Note that by default, the account will only be unlocked for 5 minutes (300s). 
#   Specify a different duration in the geth personal.unlockAccount('acct','passwd',300) call, or 0 for no limit

compiled = compile_source(source)
compiledCode = compiled['<stdin>:Example']['bin']
compiledCode = '0x'+compiledCode # This is a hack which makes the system work
```

If there are errors at this point, validate that the earlier steps
were done correctly.

* Submit Contract

```python
# Put the contract in the pool for mining, with a gas reward for processing
contractTx = c.create_contract(c.eth_coinbase(), compiledCode, gas=300000)
print("Contract transaction id is "+contractTx)
```

The result of the `print` will be something like:

```python
Contract address is 0x24be76eccfe223268e808511b5623eb8a90f91b0
```

* Busy wait

```python
print("Waiting for the contract to be mined into the blockchain...")
while c.eth_getTransactionReceipt(contractTx) is None:
        time.sleep(1)
```

* Display the contract details

```python
contractAddr = c.get_contract_address(contractTx)
print("Contract address is "+contractAddr)

# There is no cost or delay for reading the state of the blockchain, as this is held on our node
results = c.call(contractAddr, 'get_s()', [], ['string'])
print("The message reads: '"+results[0]+"'")
```

Output from the `print` will be:

```python
The message reads: 'Hello World!'
```

* Interact with the contract

```python
# Send a transaction to interact with the contract
#   We are targeting the set_s() function, which accepts 1 argument (a string)
#   In the contact definition, this was   function set_s(string new_s){s = new_s;}
params = ['Hello, fair parishioner!']
tx = c.call_with_transaction(c.eth_coinbase(), contractAddr, 'set_s(string)', params)

print("Sending these parameters to the function: "+str(params))
print("Waiting for the state changes to be mined and take effect...")
while c.eth_getTransactionReceipt(tx) is None:
    time.sleep(1)

results = c.call(contractAddr, 'get_s()', [], ['string'])
print("The message now reads '"+results[0]+"'")
```

Output here is:

```python
Sending these parameters to the function: ['Hello, fair parishioner!']
The message now reads 'Hello, fair parishioner!'
```

### Anaconda for solver

```
conda install -c cvxgrp ecos scs multiprocess
pip install cvxpy 
pip install cvxopt
```

### Truffle (JavaScript for Ethereum)

```
sudo npm install -g truffle 
```

