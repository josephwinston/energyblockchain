# Getting Started

This document shows how to use `testrpc` with smart contract.

## Installation

## Adding testrpc (Linux/mac os)

* `sudo npm install -g ethereumjs-testrpc`

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

* `testrpc` in its own terminal

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

