---
title: "Register as a Validator"
linkTitle: "Register Validator"
weight: 10
description: >
  How to register your node as a Validator on an Autonity network
---

## Prerequisites

To register a validator you need:

- A [running instance of the Autonity Go Client](/node-operators/) running on your host machine, with [networking](/node-operators/install-aut/#network) configured to allow incoming traffic on its WebSocket port.  This will be the node to be registered as a validator.
- A [running instance of the Autonity Oracle Server](/oracle/) running on your host machine, with a funded oracle server account. This will be configured to provide data price reports to your  validator node's WebSocket port.
- A configured instance of [`aut`](/account-holders/setup-aut/).
- An [account](/account-holders//create-acct/) that has been [funded](/account-holders/fund-acct/) with auton (to pay for transaction gas costs). Note that this account will become the validator's [`treasury account`](/concepts/validator/#treasury-account) - the account used to manage the validator, that will also receive the validator's share of staking rewards.

{{< alert title="Note" >}}
See the [Validator](/concepts/validator/) section for an explanation of the validator, a description of the [validator lifecycle](/concepts/validator/#validator-lifecycle), and a description of the [post-genesis registration](/concepts/validator/#post-genesis-registration) process.

See the [Oracle](/concepts/oracle-server/) section for an explanation of the oracle server.
{{< /alert >}}

## Register as a validator

### Step 1. Generate a cryptographic proof of node ownership

This must be performed on the host machine running the Autonity Go Client, using the `autonity genOwnershipProof` command:

```bash
autonity genOwnershipProof --nodekey <NODE_KEY_PATH> --oraclekey <ORACLE_KEY_PATH> <TREASURY_ACCOUNT_ADDRESS>
```

If you are running the Autonity Go Client in a docker container, setup as described in the [Run Autonity section](../../node-operators/run-aut#run-docker) (i.e. with the `autonity-chaindata` directory mapped to a host directory of the same name), the proof can be generated as follows:

```bash
docker run -t -i --volume $(pwd)/autonity-chaindata:/autonity-chaindata --volume <ORACLE_KEY_PATH>:/oracle.key --name autonity-proof --rm ghcr.io/autonity/autonity:latest genOwnershipProof --nodekey ./autonity-chaindata/autonity/nodekey --oraclekey oracle.key <TREASURY_ACCOUNT_ADDRESS>
```

where:

  - `<NODE_KEY_PATH>`: is the path to the private key file of the P2P node key (by default within the `autonity` subfolder of the `--datadir` specified when running the node. (For setting the data directory see How to [Run Autonity](/node-operators/run-aut/).)
  - `<ORACLE_KEY_PATH>`: is the path to the private key file of the oracle server key. (For creating this key see How to [Run Autonity Oracle Server](/oracle/run-oracle/).)
  - `<TREASURY_ACCOUNT_ADDRESS>`: is the account address you will use to operate the validator and receive commission revenue rewards to (i.e. the address you are using to submit the registration transaction from the local machine).

You should see something like this:

```bash
autonity genOwnershipProof --nodekey ./autonity-chaindata/autonity/nodekey --oraclekey oracle.key 0xd4eddde5d1d0d7129a7f9c35ec55254f43b8e6d4
Signature hex: 0x116f317684203d325732fbdd74632649c60ff9b246e09aced5172a0ab87ed8014b43cdce2f4c745e7c18272bc360066ee8b737bbbf27b82f9ddcd18cdc792f29012b5c3aad85f54fb2ff530a69dbd5cb5bf27dfc1658bc6f496dba4bec7d12e65a243ec8f79a4b5fbc6913273072dd1eaddee6e3b8fb699ba9b924c7d015a9c35c00
```

This signature hex will be required for the registration.

{{< alert title="Note" >}}
The `genOwnershipProof` command options `--nodekey` and `--oraclekey` options require the raw (unencrypted) private key file is passed in as argument. The `nodekey` file is unencrypted. If `aut` has been used to generate the oracle key, then the key has been created in encrypted file format using the [Web3 Secret Storage Definition <i class='fas fa-external-link-alt'></i>](https://ethereum.org/en/developers/docs/data-structures-and-encoding/web3-secret-storage/).

Autonity's `ethkey` cmd utility can be used to inspect the keystore file and view the account address, public key, and private key after entering your account password:

```
./build/bin/ethkey inspect --private <ORACLE_KEY_PATH>/oracle.key                   

Password: 
```
To install the `cmd` utilities use `make all` when [building Autonity from source code](/node-operators/install-aut/#install-source).

{{< /alert >}}

### Step 2. Determine the validator enode and address

<!-- Seems like it should be possible to do this from the host machine with an `autonity ...` cmd. -->

Ensure that `aut` connects to the node that will become a validator.  Query the enode using the `aut node info` command:

```bash
aut node info
```
```bash
$ aut node info
{
"eth_accounts": [],
"eth_blockNumber": 113463,
"eth_gasPrice": 1000000000,
"eth_hashrate": 0,
"eth_mining": false,
"eth_syncing": false,
"eth_chainId": 65000011,
"net_listening": true,
"net_peerCount": 0,
"net_networkId": "65100000",
"web3_clientVersion": "Autonity/v0.9.0-773923af-20221021/linux-amd64/go1.18.1",
"admin_enode": "enode://c746ded15b4fa7e398a8925d8a2e4c76d9fc8007eb8a6b8ad408a18bf66266b9d03dd9aa26c902a4ac02eb465d205c0c58b6f5063963fc752806f2681287a915@51.89.151.55:30303",
"admin_id": "f8d35fa6019628963668e868a9f070101236476fe077f4a058c0c22e81b8a6c9"
}
```

The url is returned in the `admin_enode` field.

The [validator address](/concepts/validator/#validator-identifier) or [validator identifier](/concepts/validator/#validator-identifier) is derived from the validator [P2P node key](/concepts/validator/#p2p-node-key)'s public key.  It can be computed from the enode string before registration:

```bash
aut validator compute-address enode://c746ded15b4fa7e398a8925d8a2e4c76d9fc8007eb8a6b8ad408a18bf66266b9d03dd9aa26c902a4ac02eb465d205c0c58b6f5063963fc752806f2681287a915@51.89.151.55:30303
```
```bash
0x49454f01a8F1Fbab21785a57114Ed955212006be
```

Make a note of this identifier.

### Step 3. Submit the registration transaction

{{< alert title="Important Note" >}}
The commands given in this step assume that your `.autrc` configuration file contains a `keyfile = <path>` entry pointing to the keyfile for the treasury account used to generate the proof of node ownership above.  If this is not the case, use the `--keyfile` option in the `aut validator register` and `aut tx sign` command below, to ensure that the registration transaction is compatible with the proof.
{{< /alert >}}

```bash
aut validator register <ENODE_URL> <ORACLE_ADDRESS> <PROOF> | aut tx sign - | aut tx send -
```

where:

- `<ENODE_URL>`: the enode url returned in Step 2.
- `<ORACLE_ADDRESS>`: the oracle server account address.
- `<PROOF>`: the proof of node ownership generated in Step 1.

Once the transaction is finalized (use `aut tx wait <txid>` to wait for it to be included in a block and return the status), the node is registered as a validator in the active state. It will become eligible for [selection to the consensus committee](/concepts/validator/#eligibility-for-selection-to-consensus-committee) once stake has been bonded to it.

#### Troubleshooting

Errors of the form
```bash
Error: execution reverted: Invalid proof provided for registration
```
indicate a mismatch between treasury address and either:
<!--
- the `from` address of the transaction generated in the `aut validator register` command, AND/OR

-->
- the `from` address of the transaction generated in the `aut contract tx` command, AND/OR
- the key used in the `aut tx sign` command

Check your configuration as described in the "Important Note" at the start of this section.

### Step 4. Confirm registration

Confirm that the validator has been registered by checking that its identifier (noted in Step 2) appears in the validator list:
```bash
aut validator list
```
```bash
0x32F3493Ef14c28419a98Ff20dE8A033cf9e6aB97
0x31870f96212787D181B3B2771F58AF2BeD0019Aa
0xE03D1DE3A2Fb5FEc85041655F218f18c9d4dac55
0x52b89AFA0D1dEe274bb5e4395eE102AaFbF372EA
0x49454f01a8F1Fbab21785a57114Ed955212006be
```

Confirm the validator details using:

```bash
aut validator info --validator 0x49454f01a8F1Fbab21785a57114Ed955212006be
```

and check that the information is as expected for your validator.

{{% pageinfo %}}
To self-bond stake to your validator node, submit a bond transaction from the account used to submit the registration transaction - i.e. the validator's treasury account address. For how to  do this see the how to [Bond stake](/delegators/bond-stake/).
{{% /pageinfo %}}
