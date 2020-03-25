# Nightlite

This library strips down the main Nightfall repository to the minimum needed to run the Nightfall
protocol on other applications.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Installation and Configuration](#installation-and-configuration)
- [Trusted Setup](#trusted-setup)
- [ZKP Public/Private Keys](#zkp-publicprivate-keys)
- [Deploy Necessary Contracts](#deploy-necessary-contracts)
- [Deploy VKs to the blockchain](#deploy-vks-to-the-blockchain)
- [Run Nightfall Functions](#run-nightfall-functions)
- [To Do](#to-do)
  - [Passing Providers](#passing-providers)
  - [Acknowledgements](#acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Installation and Configuration

To install nightlite, run `npm install --save @eyblockchain/nightlite`

These instructions assume that you're running your application on a Dockerized Linux container. The
reason for this is because Zokrates is required to run on Linux. Nightlite can presumably be run
natively on Linux, but this guide will not provide support or instruction for that.

In your Dockerfile that will be running `nightlite`, you will need to perform a multi-stage build.
Import the official ZoKrates image like so:

```Dockerfile
# Pull in a Zokrates container so that we can pull its contents into the below container.
FROM zokrates/zokrates:0.5.1 as builder
```

And then, (assuming this is your final Docker container), copy the necessary Zokrates files into
your container:

```Dockerfile
FROM node:11.15 WORKDIR /app

# Copy over Zokrates files into this container
COPY --from=builder /home/zokrates/zokrates /app/zokrates
COPY --from=builder /home/zokrates/.zokrates\* /app/stdlib
```

`Nightlite` defaults logging levels to `info`, but if you want more detailed console logs, you can
set the logging level to `verbose`, or for even more detail, `debug` by setting an environment
variable `NIGHTLITE_LOG_LEVEL` to DEBUG

Finally, on startup, your application should run `provider.connect(<ProviderURL>)`. (e.g.,
`provider.connect('ws://ganache:8545')`) This will set the provider that all Nightfall smart
contract calls will use.

## Trusted Setup

The `setup/gm17` directory contains the .zok files that you need in order to run the Nightfall
functions. The `generateZokratesFiles()` function completes the setup for you, skipping the `common`
folder, containing shared functions, and the `mimc` folder, containing the equivalent .zok files
which use MiMC hashing.

All the .zok files need to have the `generateZokratesFiles()` function run on them. This will
generate the files you need for the rest of the Nightfall protocol to work. If you are running
Nightfall, the `./nightfall-generate-trusted-setup` command calls this function for you.

Otherwise, `generateZokratesFiles()` requires a directory argument telling it where to output the
files. It can take a second optional argument telling it which file to set up. For example:

```sh
generateZokratesFiles('zkp/gm17', 'ft-transfer')
```

will set up only `ft-transfer.zok` and output the files in your `zkp/gm17` directory. If `HASH_TYPE`
is set to `mimc`, the function will automatically set up `mimc/ft-transfer.zok`.

By default, Nightlite will use SHA-256 for merkle tree calculations. If you would like to use MiMC
instead, change the `HASH_TYPE` value in `config.js` and set up the files as normal.

Using Nightfall, you can choose whether to use MiMC hashing or SHA-256 hashing within the
application. It will automatically set up the correct files for you. Take a look at the
[Nightfall](https://github.com/EYBlockchain/nightfall/tree/master/zkp) `zkp/README` for more
information.

Similarly, you can set the COMPLIANCE environment variable to 'true'. This will set up the code to
work with the circuits in `setup/gm17/rc`, which will enable blacklisting of users and decryption of
transactions.

The Trusted Setup step will take approximately one hour. The Trusted Setup step will need to be
re-run for a given .zok file whenever it is changed or whenever you change `HASH_TYPE`.

## ZKP Public/Private Keys

In order to make private transactions, you will need a ZKP public/private key pair. This is separate
from the typical Ethereum public/private key pair.

The ZKP public/private keys are both 32 bytes long. As a string, this a 66 character value (0x + 64
characters).

You can generate a private key by generating any random 32 byte string (you can use our
`utils.randomHex(32)` function).

You can generate your matching public key by hashing it (you can use our `utils.hash()` function).

Just as with typical Ethereum key pairs, losing your private key can mean the loss of any
commitments you hold.

## Deploy Necessary Contracts

The following contracts are necessary for Nightfall:

- Verifier_Registry
- BN256G2
- GM17_v0
- FToken
- FTokenShield
- NFTokenMetadata
- NFTokenShield

The deployment currently occurs in `zkp/migrations/2_Shield_migration.js`. We may move away from
truffle deployments and use web3 or another similar library in the future.

FToken and NFTokenMetadata are placeholder ERC721/ERC20 contracts. In order to replace them, you
need to swap the FToken/NFTokenMetadata contracts in this migration script.

## Deploy VKs to the blockchain

The Verification Keys that we generated earlier in the `Trusted Setup` step need to be deployed to
the blockchain. We deploy them directly to the Shield Contracts. The function `loadVk()` loads the
`vk.json` files we made in the Trusted Setup stage to the Shield contract(s).

`loadVk()` must be called on each `vk.json`. Those VKs must then be uploaded to the FTokenShield and
NFTokenShield contracts via their `registerVerificationKey()` functions. The Shield contract keeps
track of which verification key relates to which function (e.g. it stores which verification key
relates to a 'transfer').

A sample implementation can be found in Nightfall's `zkp/src/vk-controller.js`, in the function
`initializeVks()`.

## Run Nightfall Functions

There are currently six Nightfall functions, `Mint`, `Transfer`, and `Burn` for both ERC20 and
ERC721 contracts. After the above steps are completed, you can call those functions as many times as
you'd like. The above steps do not need to be repeated (assuming your environment is now setup).

Note that there are certain things that need to be stored while running these functions.

When a commitment is generated (whether its through minting a commitment, or `ft-transfer`'s
"change" mechanic), it has a `salt`, a `commitment`, and a `commitmentIndex`. All of these things
are required for later function calls. Refer to the documentation on each individual function for
more information.

A consolidation transfer (`ft-consolidation-transfer`), which takes 20 commitments and sends them in
one proof, is only possible with MiMC hashing due to its efficiency in ZKP circuits. If you would
like to use it, or MiMC hashing in general, be sure to
[re-run](https://github.com/EYBlockchain/nightfall/tree/master/zkp) the trusted setup on the files
in `gm17/mimc`. This can be done by changing the `HASH_TYPE` variable to `'mimc'` and proceeding as
normal.

## To Do

### Passing Providers

Currently, most functions that interact with smart contracts just "know" what the proper provider
is, but this isn't good. We need to figure out how to get these functions their providers.

Here are some possibilities:

1. **Pass the provider to each function**: The most straightforward, but also a lot of clutter
2. Set a "provider" singleton: Requires some additional setup from the user (probably just calling
   `setProvider()` on startup).

### Acknowledgements

Team Nightfall thanks those who have indirectly contributed to it, with the ideas and tools that
they have shared with the community:

- [ZoKrates](https://hub.docker.com/r/michaelconnor/zok)
- [Libsnark](https://github.com/scipr-lab/libsnark)
- [Zcash](https://github.com/zcash/zcash)
- [GM17](https://eprint.iacr.org/2017/540.pdf)
- [0xcert](https://github.com/0xcert/ethereum-erc721/)
- [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-solidity/blob/master/contracts/token/ERC20/ERC20.sol)

Thanks to John Sterlacci for the name `Nightlite`.