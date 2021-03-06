<!--
order: 5
-->

# Service Providers

We define 'service providers' as entities providing services for end-users that involve some form of interaction with a OKChain-SDK based blockchain (this includes the OKChain). More specifically, this document will be focused around interactions with tokens.

This section does not concern wallet builders that want to provide Light-Client functionalities. Service providers are expected to act as trusted point of contact to the blockchain for their end-users. 

## High-level description of the architecture

There are three main pieces to consider:

- Full-nodes: To interact with the blockchain. 
- Rest Server: This acts as a relayer for HTTP calls.
- Rest API: Define available endpoints for the Rest Server.

## Running a Full-Node

### Installation and configuration

We will describe the steps to run and interact with a full-node for the OKChain. For other SDK-based blockchain, the process should be similar. 

First, you need to [install the software](../okchain-tutorials/installation.md).

Then, you can start [running a full-node](../okchain-tutorials/join-mainnet.md).

### Command-Line interface

## Setting Up `okchaincli`

::: tip
**Before setting up `okchaincli`, make sure you have set up a way to [access the OKChain network](#accessing-the-cosmos-hub-network)**
:::

::: warning
**Please check that you are always using the latest stable release of `okchaincli`**
:::

`okchaincli` is the tool that enables you to interact with the node that runs on the OKChain network, whether you run it yourself or not. Let us set it up properly.

In order to set up `okchaincli`, use the following command:

```bash
okchaincli config <flag> <value>
```

It allows you to set a default value for each given flag. 

First, set up the address of the full-node you want to connect to:

```bash
okchaincli config node <host>:<port

// example: okchaincli config node https://77.87.106.33:26657
```

If you run your own full-node, just use `tcp://localhost:26657` as the address. 

Then, let us set the default value of the `--trust-node` flag:

```bash
okchaincli config trust-node false

// Set to true if you run a light-client node, false otherwise
```

Finally, let us set the `chain-id` of the blockchain we want to interact with:

```bash
okchaincli config chain-id cosmoshub-2
```

Next you will find a few useful CLI commands to interact with the Full-Node.

#### Creating a key-pair

To generate a new key (default secp256k1 elliptic curve):

```bash
okchaincli keys add <your_key_name>
```

You will be asked to create a password (at least 8 characters) for this key-pair. This will return the information listed below:

- `NAME`: Name of your key
- `TYPE`: Type of your key, always `local`. 
- `ADDRESS`: Your address. Used to receive funds.
- `PUBKEY`: Your public key. Useful for validators.
- `MNEMONIC`: 24-words phrase. **Save this mnemonic somewhere safe**. It is used to recover your private key in case you forget the password.

You can see all your available keys by typing:

```bash
okchaincli keys list
```

#### Checking your balance

After receiving tokens to your address, you can view your account's balance by typing:

```bash
okchaincli query account <YOUR_ADDRESS>
```

*Note: When you query an account balance with zero tokens, you will get this error: No account with address <YOUR_ADDRESS> was found in the state. This is expected! We're working on improving our error messages.*

#### Sending coins via the CLI

Here is the command to send coins via the CLI:

```bash
okchaincli tx send <from_key_or_address> <to_address> <amount> \
    --chain-id=<name_of_testnet_chain> 
```

Parameters:

- `<from_key_or_address>`: Key name or address of sending account.
- `<to_address>`: Address of the recipient.
- `<amount>`: This parameter accepts the format `<value|coinName>`, such as `10faucetToken`.

Flags:

- `--chain-id`: This flag allows you to specify the id of the chain. There will be different ids for different testnet chains and main chain.

#### Help

If you need to do something else, the best command you can run is:

```bash
okchaincli 
```

It will display all the available commands. For each command, you can use the `--help` flag to get further information. 

## Setting up the Rest Server

The Rest Server acts as an intermediary between the front-end and the full-node. You don't need to run the Rest Server on the same machine as the full-node. 

To start the Rest server: 

```bash
okchaincli rest-server --node=<full_node_address:full_node_port>
```

Flags:
- `--trust-node`: A boolean. If `true`, light-client verification is disabled. If `false`, it is disabled. For service providers, this should be set to `true`. By default, it set to `true`. 
- `--node`: This is where you indicate the address and the port of your full-node. The format is `<full_node_address:full_node_port>`. If the full-node is on the same machine, the address should be `tcp://localhost:26657`.
- `--laddr`: This flag allows you to specify the address and port for the Rest Server (default `1317`). You will mostly use this flag only to specify the port, in which case just input "localhost" for the address. The format is <rest_server_address:port>.


### Listening for incoming transaction

The recommended way to listen for incoming transaction is to periodically query the blockchain through the following endpoint of the LCD:

[`/bank/balance/{address}`](https://cosmos.network/rpc/#/ICS20/get_bank_balances__address_)

## Rest API

The Rest API documents all the available endpoints that you can use to interact
with your full node. It can be found [here](https://cosmos.network/rpc/).

The API is divided into ICS standards for each category of endpoints. For
example, the [ICS20](https://cosmos.network/rpc/#/ICS20/) describes the API to
interact with tokens.

To give more flexibility to developers, we have included the ability to
generate unsigned transactions, [sign](https://cosmos.network/rpc/#/ICS20/post_tx_sign)
and [broadcast](https://cosmos.network/rpc/#/ICS20/post_tx_broadcast) them with
different API endpoints. This allows service providers to use their own signing
mechanism for instance.

In order to generate an unsigned transaction (example with
[coin transfer](https://cosmos.network/rpc/#/ICS20/post_bank_accounts__address__transfers)),
you need to use the field `generate_only` in the body of `base_req`.

## OKChain SDK Transaction Signing

OKChain SDK transaction signing is a fairly simple process.

Every OKChain SDK transaction has a canonical JSON representation. The `okchaincli`
and Stargate REST interfaces provide canonical JSON representations of transactions
and their "broadcast" functions will provide compact Amino (a protobuf-like wire format)
encoding translations.

Things to know when signing messages:

The format is as follows

```json
{
  "account_number": XXX,
  "chain_id": XXX,
  "fee": XXX,
  "sequence": XXX,
  "memo": XXX,
  "msgs": XXX
}
```

The signer must supply `"chain_id"`, `"account number"` and `"sequence number"`.

The `"fee"`, `"msgs"` and `"memo"` fields will be supplied by the transaction
composer interface.

The `"account_number"` and `"sequence"` fields can be queried directly from the
blockchain or cached locally. Getting these numbers wrong, along with the chainID,
is a common cause of invalid signature error. You can load the mempool of a full
node or validator with a sequence of uncommitted transactions with incrementing
sequence numbers and it will mostly do the correct thing.  

Before signing, all keys are lexicographically sorted and all white space is
removed from the JSON output.

The signature encoding is the 64-byte concatenation of ECDSArands (i.e. `r || s`),
where `s` is lexicographically less than its inverse in order to prevent malleability.
This is like Ethereum, but without the extra byte for PubKey recovery, since
Tendermint assumes the PubKey is always provided anyway.

Signatures and public key examples in a signed transaction:

``` json
{
  "type": "auth/StdTx",
  "value": {
    "msg": [...],
    "signatures": [
      {
        "pub_key": {
          "type": "tendermint/PubKeySecp256k1",
          "value": XXX
        },
        "signature": XXX
      }
    ],
  }
}
```

Once signatures are properly generated, insert the JSON into into the generated
transaction and then use the broadcast endpoint.
