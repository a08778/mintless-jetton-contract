# Mintless Jetton

This repository contains the reference implementation of the [TEP#177](https://github.com/ton-blockchain/TEPs/pull/177) with [TEP#176](https://github.com/ton-blockchain/TEPs/pull/176) API endpoint, an extension to the [TEP#74](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md) Jetton standard. It allows for Merkle-proof airdrops, enabling the minting of Jettons directly on the Jetton-Wallet contract in a decentralized manner. The implementation is designed to support large-scale airdrops without incurring high costs or placing significant load on the blockchain.

## Features

- **Merkle Proof Airdrops**: Efficiently airdrop to millions of users using a single hash storage.
- **Decentralized Claiming**: Users can claim their airdrops through a simple transaction, which mints the Jettons on demand.
- **Integration Friendly**: Designed to be transparent for user given proper support from wallets: all magic happen under the hood while user interact with unclaimed jetton the same way as with usual.

## Prior art
Made from [Jetton with governance(stablecoin)](https://github.com/ton-blockchain/stablecoin-contract) by removing governance functionality.
Also burning is allowed.

## Local Development

### Install Dependencies

`npm install`

### Compile Contracts

`npm run build`

### Run Tests

`npm run test`

### Deploy or run another script

`npx blueprint run` or `yarn blueprint run`

use Toncenter API:

`npx blueprint run --custom https://testnet.toncenter.com/api/v2/ --custom-version v2 --custom-type testnet --custom-key <API_KEY> `

API_KEY can be obtained on https://toncenter.com or https://testnet.toncenter.com

## Deployment

> ⚠️ **Important notice:**
> The jetton wallet in this project makes use of [library cell](https://docs.ton.org/v3/documentation/data-formats/tlb/library-cells#introduction) for code storage.
> And execution guarantee is **ONLY** valid if wallet is deployed as library.
> If deployed otherwise, execution guarantee is a matter of current network configuration.

Library deployment reduces the cont of the wallet storage and
deployment stage, as the code
cell in [StateInit](https://github.com/ton-blockchain/ton/blob/cac968f77dfa5a14e63db40190bda549f0eaf746/crypto/block/block.tlb#L144) is represented by it's hash instead of full content.

However, certain conditions must be satisfied for this to work:

- The library has to be deployed on the network.
- Jetton minter must have wallet code as a library cell to utilize it.

### Library deployment

The library for the current code is already deployed by the TON Core team.

This means that if no modifications applied to the current code, developers can utilize publicly available library
without needing to deploy a new one.

If **ANY** changes are made to the jetton-wallet code, or source files it [includes](https://github.com/ton-blockchain/mintless-jetton-contract/blob/77763e6b2df6cf96b0840c7306844751200ff046/contracts/jetton-wallet.fc#L5-L10)
developer has two options:

- Deploy a new library to the network
- Re-calculate [gas constants](https://github.com/ton-blockchain/mintless-jetton-contract/blob/main/contracts/gas.fc) for usage without library cell

In order to deploy a library to the network, [librarian](https://github.com/ton-blockchain/mintless-jetton-contract/blob/main/contracts/helpers/librarian.func) contract is used
along with [deployLibrary](https://github.com/ton-blockchain/mintless-jetton-contract/blob/main/scripts/deployLibrary.ts) script.

The current default library [storage duration](https://github.com/ton-blockchain/mintless-jetton-contract/blob/77763e6b2df6cf96b0840c7306844751200ff046/contracts/helpers/librarian.func#L5) is set to 100 years and would cost roughly about **700 TON** depending on the resulting code size.

Developers are free to change the duration directly in the librarian contract,
however, it is crucial to understand the implications of library expiry.

If library cell storage expires, the wallet code becomes inaccessible,
meaning wallets will not be able
to process any further transactions.
Therefore, it is the developer responsibility to deploy the library for
a reasonable time period and monitor it's expiration date.

To extend the library storage, top-up the previous librarian address or repeat the library deployment procedure.

### Deploying Minter with library wallet code

In current repository setup minter is deployed with library wallet
code automatically by the deployMinter script.
However, some developers may use the contract code in a
different environment, so it's important to
ensure that the minter wallet code configuration is valid.

Process of deploying minter with wallet code as library is fairly straightforward:

1. Get library cell from the ordinary code cell
2. Pass it as a wallet code to your minter constructor
3. Deploy the minter

For the first step, refer [this](https://docs.ton.org/v3/documentation/data-formats/tlb/library-cells#store-data-in-a-library-cell)
chapter of documentation, which illustrates how to accomplish this
using `@ton/ton` or Fift language.
Use the provided examples to adapt it to the environment of choice.

### Check deployment validity

In order to determine if library has been deployed successfully:

1. Find your librarian contract address in the explorer of choice
2. Check that account status is active and balance is positive. If deployment failed, contract should return all incoming value to the sender
3. Check that account code and data are set empty cell `x{}` or `96a296d224f285c67bee93c30f8a309157f0daa35dc5b87e410b78630a09cfc7` in hash view
4. Take your **RAW** code hash, and pass it in upper case to [get_lib](https://dton.io/graphql/#query=%7B%0A%20%20get_lib(lib_hash%3A%20%220F1AD3D8A46BD283321DDE639195FB72602E9B31B1727FECC25E2EDC10966DF4%22)%0A%7D)
5. `get_lib` result should be non-empty

> ⚠️ **Important notice**
Before performing any token mint, it is highly recommended
to perform wallet code check right after the minter deployment,
regardless of the deployment environment

In order to do so, run
`npx blueprint run checkWalletLib`

### Re-calculating gas constants of library-less deployment

Please refer to [DEVELOPMENT.md](https://github.com/ton-blockchain/mintless-jetton-contract/blob/main/DEVELOPMENT.md) for the information on gas constats.

## Examples

Check [generateTestJetton](./scripts/generateTestJetton.ts) as example of deploying mintless jetton and [claimApi.ts](./scripts/claimApi.ts) for example api endpoint. Note, `claimAPI.ts` is not intended to be used for millions of users, check https://github.com/Trinketer22/proof-machine for mass-scale example.
