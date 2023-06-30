# Nouns DAO audit details

- Total Prize Pool: $100,000 USDC
  - HM awards: $69,712.50 USDC
  - Analysis awards: $4,225 USDC
  - QA awards: $2,112.50 USDC
  - Bot Race awards: $6,337.50 USDC
  - Gas awards: $2,112.50 USDC
  - Judge awards: $9,000 USDC
  - Lookout awards: $6,000 USDC
  - Scout awards: $500 USDC
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2023-06-nouns-dao/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts July 03, 2023 20:00 UTC
- Ends July 13, 2023 20:00 UTC

## Automated Findings / Publicly Known Issues

Automated findings output for the audit can be found [here](add link to report) within 24 hours of audit opening.

_Note for C4 wardens: Anything included in the automated findings output is considered a publicly known issue and is ineligible for awards._

# About Nouns DAO

[Nouns](https://nouns.wtf/) is a generative NFT project on Ethereum, where a new Noun is minted and auctioned off every day, all proceeds go to the DAO treasury, and each token represents one vote. To date more than 720 Nouns have been sold on auction, and the DAO has funded a variety of [coders](https://nouns.wtf/vote/154), [artists](https://youtu.be/oa79nN4gMPs), [athletes](https://youtu.be/JQSmfSnRGVk), and IRL and [Ethereum](https://nouns.wtf/vote/249) public goods.

More intro information can be found on [Nouns Center](https://nouns.center/).

# Project Intro

TLDR: we're upgrading our governor contract with a bunch of new features, including [Nouns Fork](https://mirror.xyz/0x10072dfc23771101dC042fD0014f263316a6E400/iN0FKOn_oYVBzlJkwPwK2mhzaeL8K2-W80u82F7fHj8), a minority protection mechanism that allows a group of token holders to exit together into a new instance of Nouns DAO.

## Nouns contracts

Nouns DAO relies on a handful of contracts on mainnet today:

1. [Token](https://etherscan.io/address/0x9c8ff314c9bc7f6e59a9d9225fb22946427edc03#code): a fork of Compound's Comp, modified to ERC721 instead of ERC20.
2. [Governor](https://etherscan.io/address/0x6f3E6272A167e8AcCb32072d08E0957F9c79223d#code): originally a fork of Compound's Governor Bravo, a transparent proxy currently pointing to [DAO Logic V2](https://etherscan.io/address/0x51C7D7C47E440d937208bD987140D6db6B1E4051#code).
3. [Executor](https://etherscan.io/address/0x0bc3807ec262cb779b38d65b38158acc3bfede10#code): the DAO's treasury, a fork of Compound's Timelock.
4. [Auction House](https://etherscan.io/address/0x830BD73E4184ceF73443C15111a1DF14e495C706#code): a fork of Zora's AuctionHouse, a transparent proxy pointing to [the original logic](https://etherscan.io/address/0xf15a943787014461d94da08ad4040f79cd7c124e#code) Nouns launched with.
5. [TokenBuyer](https://etherscan.io/address/0x4f2acdc74f6941390d9b1804fabc3e780388cfe5#code): a contract that swap ETH for USDC asynchronously, helping the DAO pay proposal builders without volatility risk.
6. [Streamer](https://etherscan.io/address/0x0fd206fc7a7dbcd5661157edcb1ffdd0d02a61ff#code): a fork of Sablier, allowing the DAO to create streams and fund them asynchronously, e.g. using TokenBuyer.

## Change summary

1. **Proposal editing**: allowing proposers to update their proposal's transactions and text description, during the Updatable period only, which is the state upon proposal creation. Editing also works with signatures, assuming the proposer is able to accumulate signatures from the same signers.
2. **Propose by signature**: allowing Nouners and delegates to pool their voting power towards submitting a proposal, by submitting their signature, instead of the current approach where sponsors must delegate their votes to help a proposer achieve threshold.
3. **Objection-only Period**: a conditional voting period that gets activated upon a last-minute proposal swing from defeated to successful, affording against voters more reaction time. Only against votes are possible during the objection period.
4. **Votes Snapshot After Voting Delay**: moving votes snapshot up, to provide Nouners with reaction time per proposal, to get their votes ready (e.g. some might want to move their delegations around). Currently the vote snapshot block is the proposal creation block.
5. **Nouns Fork**: any token holder can signal to fork (exit) in response to a governance proposal. If a quorum of 20% of tokens signals to exit, the fork will succeed. For a deeper understanding please read [the tech spec](https://github.com/verbsteam/nouns-fork-spec/blob/main/spec.md) and [this post](https://mirror.xyz/0x10072dfc23771101dC042fD0014f263316a6E400/iN0FKOn_oYVBzlJkwPwK2mhzaeL8K2-W80u82F7fHj8).

For more detail on all features except Nouns Fork, [see this spec](https://hackmd.io/@el4d/nouns-dao-v3-spec).

# Upgrade Plan

The Nouns governor is currently at V2. This version under audit is V3, and the upgrade can only happen through a proposal, since only the treasury (executor) contract has the permission to set the governor proxy's implementation address.

The Nouns Fork feature introduced in this version requires the ability to send DAO funds to fork DAOs without going through a proposal process; this requirement means we need a new treasury, and since the current treasury is not upgradable, our plan is to migrate funds to the new treasury, and make the new treasury upgradeable to minimize the risk of another such migration in future.

Furthermore, for several reasons it's important to maintain the ability to execute proposals on the original treasury, and for that reason we've added proposal functions that would still use it on the new governor version, e.g. `proposeOnTimelockV1` and `executeOnTimelockV1`.

The deployment consists of the following scripts:

1. [`DeployDAOV3NewContractsMainnet`] Deploy all the new contracts.
2. [`ProposeDAOV3UpgradeMainnet`] Propose the upgrade to V3 and all the config changes this upgrade entails:
   1. Send ETH to treasury v2; sending a fixed amount that is lower than the entire balance, to account for proposals that might be executing on treasury V1 around the same time.
   2. Set governor proxy's implementation to V3.
   3. Set V3's fork parameters.
   4. Set the proposal ID at which vote snapshot blocks change from proposal creation block to voting period start block.
   5. Set the auction house owner to treasury v2.
   6. Approve a ERC20Transferer helper contract to transfer the DAO's stETH tokens.
   7. Call the transferer's function to move all stETH tokens from treasury v1 to v2.
   8. Transfer ownership of NounsToken to treasury V2.
   9. Transfer ownership of NounsToken's descriptor to treasury V2.
   10. Set the governor's admin and treasury addresses, making treasury v2 the main executor contract and the governor's admin.
3. [`ProposeTimelockMigrationCleanupMainnet`] Propose additional txs to treasury V1. This proposal will be proposed once `ProposeDAOV3UpgradeMainnet` has been successfully executed. This gives time for proposals using TokenBuyer (asking for USDC)
   to execute successfully.
   This script includes the following:
   1. Transfer ownership of AuctionHouse's proxy admin to treasury V2.
   2. Transfer ETH leftover from the upgrade proposal to treasury V2.
   3. Approve treasury V2 to transfer LilNouns NFTs owned by treasury V1.
   4. Transfer Noun #687 from treasury v1 to v2.
   5. Transfer TokenBuyer ownership to treasury v2.
   6. Transfer Payer ownership to treasury v2 (Payer is a helper contract part of the TokenBuyer).
4. [`ProposeENSReverseLookupConfigMainnet`] Propose post-upgrade on treasury V2 to set its ENS reverse lookup to `nouns.eth`.
   1. Worth noting this proposal should only be executed after the owner of `nouns.eth` updates the main record to point to treasury V2 instead of V1. The relevant person from the founders is ready to execute the ENS changes in time.

# Audit Scope

The key risks we'd like for you to dig into are:

1. Bricking the DAO: might this version lead to token holders not being able to pass any more proposals?
2. Security: does this version open up ways for funds or tokens to be stolen from the DAO or its token holders?
3. Are there DAO bricking or security risks in the contracts we deploy for new Nouns Fork DAOs?

Main contracts to pay attention to:

- The new DAO governor `NounsDAOLogicV3`, including the upgrade from `NounsDAOLogicV2` to `NounsDAOLogicV3`, keeping in mind there will be proposals in progress in various states when the upgrade is executed.
- The fork escrow `NounsDAOForkEscrow`, making sure no token loss or theft is possible, including by the DAO itself, keeping in mind that forking might be used in attack scenarios.
- The new treasury `NounsDAOExecutorV2`, paying attention to the transition from the current version to this new version, and extra attention to security since this is the contract that holds all the DAO's funds.
- All the fork DAO contracts under `governance/fork/newdao/`, making sure the original DAO can't attack a fork DAO in any material way (there are known issues we're choosing to keep for other considerations, see more info below).

## Files in Scope

Please refer to [this commit](https://github.com/nounsDAO/nouns-monorepo/tree/718211e063d511eeda1084710f6a682955e80dcb) on the nouns monorepo for the version to audit.

| File                                                                            | Lines    | nLines   | nSLOC    | Comment Lines |
| ------------------------------------------------------------------------------- | -------- | -------- | -------- | ------------- |
| script/ProposeTimelockMigrationCleanupMainnet.s.sol                             | 102      | 94       | 68       | 12            |
| script/DeployDAOV3NewContractsTestnet.s.sol                                     | 39       | 39       | 33       | 3             |
| script/DeployDAOV3DataContractsSepolia.s.sol                                    | 12       | 12       | 8        | 1             |
| script/DeployDAOV3DataContractsGoerli.s.sol                                     | 12       | 12       | 8        | 1             |
| script/ProposeDAOV3UpgradeMainnet.s.sol                                         | 147      | 137      | 106      | 9             |
| script/DeployDAOV3NewContractsBase.s.sol                                        | 107      | 89       | 75       | 4             |
| script/ProposeENSReverseLookupConfigMainnet.s.sol                               | 48       | 44       | 29       | 6             |
| script/DeployDAOV3NewContractsMainnet.s.sol                                     | 22       | 22       | 18       | 3             |
| script/ProposeDAOV3UpgradeTestnet.s.sol                                         | 173      | 163      | 134      | 6             |
| script/DeployDAOV3DataContractsBase.s.sol                                       | 37       | 37       | 27       | 1             |
| contracts/utils/ERC20Transferer.sol                                             | 37       | 37       | 9        | 24            |
| contracts/governance/NounsDAOLogicV2.sol                                        | 1091     | 1032     | 616      | 292           |
| contracts/governance/NounsDAOProxy.sol                                          | 141      | 141      | 67       | 56            |
| contracts/governance/NounsDAOExecutorProxy.sol                                  | 28       | 28       | 5        | 18            |
| contracts/governance/NounsDAOLogicV3.sol                                        | 1036     | 932      | 336      | 484           |
| contracts/governance/NounsDAOV3Votes.sol                                        | 335      | 287      | 131      | 116           |
| contracts/governance/NounsDAOV3Admin.sol                                        | 608      | 565      | 291      | 164           |
| contracts/governance/fork/ForkDAODeployer.sol                                   | 173      | 165      | 98       | 41            |
| contracts/governance/NounsDAOExecutorV2.sol                                     | 249      | 227      | 136      | 48            |
| contracts/governance/fork/NounsDAOV3Fork.sol                                    | 260      | 232      | 114      | 76            |
| contracts/governance/NounsDAOV3Proposals.sol                                    | 1018     | 877      | 579      | 188           |
| contracts/governance/NounsDAOV3DynamicQuorum.sol                                | 164      | 148      | 85       | 43            |
| contracts/governance/NounsDAOInterfaces.sol                                     | 873      | 770      | 395      | 284           |
| contracts/governance/fork/NounsDAOForkEscrow.sol                                | 193      | 188      | 63       | 92            |
| contracts/governance/NounsDAOExecutor.sol                                       | 189      | 171      | 110      | 28            |
| contracts/governance/fork/newdao/NounsAuctionHouseFork.sol                      | 282      | 274      | 133      | 94            |
| contracts/governance/fork/newdao/governance/INounsTokenForkLike.sol             | 23       | 6        | 3        | 1             |
| contracts/governance/fork/newdao/governance/NounsDAOStorageV1Fork.sol           | 147      | 147      | 68       | 60            |
| contracts/governance/fork/newdao/governance/NounsDAOLogicV1Fork.sol             | 802      | 758      | 413      | 257           |
| contracts/governance/fork/newdao/governance/NounsDAOEventsFork.sol              | 71       | 71       | 38       | 19            |
| contracts/governance/fork/newdao/token/NounsTokenFork.sol                       | 328      | 320      | 140      | 124           |
| contracts/governance/fork/newdao/token/INounsTokenFork.sol                      | 58       | 41       | 14       | 14            |
| contracts/governance/fork/newdao/token/base/ERC721CheckpointableUpgradeable.sol | 293      | 265      | 131      | 94            |
| **Totals**                                                                      | **9098** | **8331** | **4481** | **2663**      |

Lines counted using [Solidity Metrics in VSCode](https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-metrics).

# Known Issues and Contingency Plans

## From this new version

### All issues documented by Spearbit

Please see [this Spearbit document](https://gist.github.com/hrkrshnn/25e13d04fa728309370acb6ca8ccc16b#file-nouns-md).

### The original DAO can temporarily brick a fork DAO's token minting

We've decided in this version to deploy fork DAOs such that fork tokens reuse the same descriptor contract as the original DAO's token descriptor. Our motivations are minimizing lines of code and the gas cost of deploying a fork.

This design poses a risk on fork tokens: the original DAO can update the descriptor to use a new art contract that always reverts, which would then lead to fork token's mint function always reverting.

The solution would be for the fork DAO to execute a proposal that deploys and sets a new descriptor to its token, which would use a valid art contract, allowing minting to resume.

The fork DAO is guaranteed to be able to propose and execute such a proposal, because the function where Nouners claim their fork tokens does not use the descriptor, and so is not vulnerable to this attack.

### Forking will fail if NounsToken's minter is not AuctionHouse

In `ForkDAODeployer.deployForDAO()`, the fork's AuctionHouse is initialized with values copied from the original Nouns AuctionHouse, by calling its getter functions. Therefore, if Nouns DAO executes a proposal that sets the token's minter to an account that does not implement the same getters, fork deployments will fail.

We view this as a minor issue, since the DAO can easily execute a proposal that switches `ForkDAODeployer` out for a new version that would support the minter change.

### When a minority forks, the majority can follow

For example, a malicious majority can vote For a proposal to drain the treasury, forcing others to fork; the majority can then join the fork with many of their tokens, benefiting from the passing proposal on the original DAO, while continuing to attack the minority in their new fork DAO, forcing them to quit the fork DAO.

This is a well known flaw of the current fork design, something we've chosen to go live with for the sake of shipping something the DAO has asked for urgently.

## From previous versions

See the [V2 audit report](https://code4rena.com/reports/2022-08-nounsdao/) for all the details.

### NounsToken delegateBySigs allows delegating to address zero

We've fixed this issue in the fork token contract, but cannot fix it in the original NounsToken because the contract isn't upgradeable.

### Voting gas refund can be abused

We're aware of different ways of abusing this function:

1. A token holder could delegate their Nouns to a contract and vote on multiple proposals in a loop, such that the tx gas overhead is amortized across all votes, while the refund function assumes each vote has the full overhead to bear; this could result in token holders profiting from gas refunds.
2. A token holder could grief the DAO by voting with very long reason strings, in order to drain the refund balance faster.

We find these issues low risk and unlikely given the small size of the community, and the low ETH balance the governor contract has to spend on refunds. Should we see such abusive activity, we might reconsider this feature.

### Nouns transfers will stop working when block number hits uint32 max value

We're aware of this issue. It means the Nouns token will stop functioning a long long long time from now :)

### AuctionHouse has an open gas griefing vector

Bidder Alice can bid from a smart contract that returns a large byte array when receiving ETH. Then if Bob outbids Alice, in his bid tx AuctionHouse refunds Alice, and the large return value causes a gas cost spike for Bob. [See more details here](https://github.com/nounsDAO/nouns-monorepo/issues/556).

We're planning to fix this in the next AuctionHouse version, its launch date is unknown at this point.

### Using error strings instead of custom errors

In all new code we're using custom errors. In code that's forked from previous versions we optimized for the smallest diff possible, and so leaving error strings untouched.

# Dev Env Setup

### Workspace setup

Clone with one of the following:

```sh
git clone --recurse-submodules git@github.com:code-423n4/2023-07-nounsdao.git
git clone --recurse-submodules https://github.com/code-423n4/2023-07-nounsdao.git
```

If the repo was cloned without recursing, it's still possible to execute the following:

```sh
git submodule update --init --recursive
```

Install the dependencies with the following:

- Run `yarn` in the `nouns-monorepo/` folder.
- Run `forge install` in the `nouns-monorepo/packages/nouns-contracts` folder.

### Running tests

All new tests use forge, with a small remainder of legacy tests in hardhat.

In the `nouns-monorepo/packages/nouns-contracts` folder:

- For forge tests run: `forge test --ffi`.
- For hardhat tests run: `yarn test`.
- For Slither run: `slither . --foundry-out-directory artifacts --filter-paths "/test/|/lib/|/script|/node_modules/"`

The forge test suite contains a mainnet fork test:

- To run it successfully set the env var `RPC_MAINNET` to an RPC like Alchemy or Infura pointing to Ethereum mainnet. It's also possible to use a public RPC with `export RPC_MAINNET="https://eth.llamarpc.com"`.
- To skip it run tests like so: `forge test --ffi --nmc UpgradeToDAOV3ForkMainnetTest`.
