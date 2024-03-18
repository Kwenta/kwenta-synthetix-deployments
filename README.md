# Kwenta Synthetix Deployments

This the Kwenta GitOps repo for deployment of the [Synthetix](https://www.github.com/synthetixio/synthetix-v3) protocol.

The following section is the Synthetix Guide to deployment, for Kwenta specific information, jump straight to [Kwenta Deployment Notes](#kwenta-deployment-notes).

## Deployment Guide

- Run `yarn cannon setup` and ensure you have a reliable IPFS url for publishing.

### Specify Upgrade

- After [publishing any new versions of the provisioned packages](https://github.com/synthetixio/synthetix-v3#deployment-guide) (`oracle-manager`, `synthetix` and `spot-market`), bump the versions throughout the cannonfiles to match.
- Add new settings and invoke actions as necessary.
- Increment the version number and update the values in the network-specific omnibus cannonfiles as desired. The main version should match synthetix version, and if it a configuration change on the same version use a dash.
  Version: 3.3.5
  Version with config changes: 3.3.5-1

### Execute Upgrade

Conduct the following process for each network:

- Perform a dry-run and confirm that the actions that would be executed by Cannon are expected:

  ```sh
  yarn cannon build omnibus-base-sepolia-andromeda.toml \
    --dry-run \
    --upgrade-from synthetix-omnibus:latest@andromeda \
    --chain-id 84532 \
    --provider-url https://base-sepolia.infura.io/v3/$INFURA_API_KEY
  ```

- Remove the dry-run option to execute the upgrade:

  ```sh
  yarn cannon build omnibus-base-sepolia-andromeda.toml \
    --upgrade-from synthetix-omnibus:latest@andromeda \
    --private-key $TESTNET_DEPLOYER_PRIVATE_KEY \
    --provider-url https://base-sepolia.infura.io/v3/$INFURA_API_KEY
  ```

- After this you can run the dry-run command again (without upgrade-from), and should see no changes

  ```sh
  yarn cannon build omnibus-base-sepolia-andromeda.toml \
    --dry-run \
    --chain-id 84532 \
    --provider-url https://base-sepolia.infura.io/v3/$INFURA_API_KEY
  ```

_The --provider-url and --private-key parameters are unnecessary if using [Frame](https://frame.sh/)_

### Finalize Release

- If you've updated the provisioned packages, verify your new contracts on Etherscan:

  ```sh
    yarn cannon verify synthetix-omnibus:3.3.5-1@andromeda --chain-id 84531 --api-key $ETHERSCAN_API_KEY
  ```

- Publish your new packages on the Cannon registry:

  ```sh
  yarn cannon publish synthetix-omnibus:3.3.5-1@andromeda \
      --chain-id 84531 \
      --private-key $MAINNET_DEPLOYER_PRIVATE_KEY \
      --tags latest,3 \
      --include-provisioned
  ```

  If you use frame:
  (_The --private-key parameter is unnecessary if using [Frame](https://frame.sh/)_)

- Commit and merge the change to this repository.
- Run the [**Export ABIs** action](https://github.com/Synthetixio/v3-abi-exporter/actions/workflows/main.yml) in the `v3-abi-exporter` repository.

## Fork-testing locally

Example based on `omnibus-base-sepolia-andromeda.toml`

**IMPORTANT** Restart Anvil node and apply upgrades after each full test suite execution because of the global system state change, which affects things like global collateral limits

1. Build locally with --dry-run

   ```sh
   yarn cannon build omnibus-base-sepolia-andromeda.toml \
     --dry-run \
     --upgrade-from synthetix-omnibus:latest@andromeda \
     --chain-id 84532 \
     --provider-url https://sepolia.base.org \
       | tee ./e2e/cannon-build.log
   ```

   or

   ```sh
   yarn build:andromeda
   ```

2. Fetch deployments and store as JSON files

   ```sh
   yarn fetch-deployments
   ```

3. Run local Anvil node for the required network.

   ```sh
   yarn cannon build omnibus-base-sepolia-andromeda.toml \
      --port 8545 \
      --keep-alive \
      --dry-run \
      --upgrade-from synthetix-omnibus:latest@andromeda \
      --chain-id 84532 \
      --provider-url https://sepolia.base.org
   ```

   or

   ```sh
   yarn start:andromeda
   ```

4. Execute tests
   ```sh
   DEBUG='e2e:*' mocha e2e/tests/omnibus-base-sepolia-andromeda.toml/*.e2e.js
   ```
   or
   ```sh
   yarn test:base-sepolia
   ```

# Kwenta Deployment Notes

## Full Deployment Process

The following steps were used to deploy a forked "Kwenta" version of the protocol:
1. Copied `omnibus-base-goerli-competition.toml` into a new file `kwenta-omnibus-base-goerli-competition.toml`
   - Note that this version was chosen for copying, because it was used for a testnet competition on Base Goerli, and hence was best suited to deploying everything needed for a successful test deployment.
2. Make critical changes to the `.toml` cannonfile:
   1. Update the name of the of the deployment to `kwenta-synthetix-omnibus` instead of `synthetix-omnibus` - a new name is needed so we can separate our system in the registry from the one synthetix have already deployed and are managing.
   2. Update `setting.target_preset` to a new value just for our system (TODO: check why I did this & if it is really necessary).
   3. Update `setting.salt` to a unique value. This is needed to avoid a collision with the already deployed system, as use of CREATE2 will cause the same addresses to be generated, without a unique salt.
   4. Update the various `owner` settings to the chosen owner for the system. Without this change, it will not be possible to administrate the system. The following settings needs updating with this change:
       - `setting.owner`
      - `setting.pool_owner`
      - `setting.ccip_router` - normally this would be the [Cross Chain Interoperability Protocol Router](https://chain.link/cross-chain), but for testing purposes we will set it as the system owner here. 
   5. Update `setting.deployer` address to the chosen deployer address. Without this change the deployment will fail. I used the same `owner` and `deployer` for simplicity (though in production it may be wise to use different addresses for security reasons)
   6. I removed `invoke.unapprove_wrongfully_deployed_snx_pool` as this was not relevant to our system.
3. Run a dry-run of the deployment to ensure it will be successful before executing against the actual chain:
```bash
cannon build kwenta-base-goerli-competition.toml --provider-url <BASE_GOERLI_RPC_URL> --private-key <DEPLOYER_PRIVATE_KEY> --dry-run
```
- `<BASE_GOERLI_RPC_URL> ` should be an RPC url for an eth node on the Base Goerli network.
- `<DEPLOYER_PRIVATE_KEY>` should be the private key for the EOA you want to deploy the system from. I used the Kwenta Testnet Admin (address: `0xC2ecD777d06FFDF8B3179286BEabF52B67E9d991`).
- This is the step to try and debug any issues with the deployment before using real ETH.
4. Send a chunky amount of ETH (it cost me 0.17 ETH, but send extra!) to the deployment address. Gas costs are volatile and you don't want to get caught with a mid-deployment failure. A semi-deployed/configured system is hard to fix! The Synthetix system is expensive to deploy. It is large and consists of many different modules and proxies. In theory cannon provides functionality for recovering a deployment that failed mid-way, in practice I was not able to get this to work.
2. Now you are ready for the real deal! Use this command to actually deploy the system 🔥 🚀 💫 (simply remove the `--dry-run` flag):
```bash
cannon build kwenta-base-goerli-competition.toml --provider-url <BASE_GOERLI_RPC_URL> --private-key <DEPLOYER_PRIVATE_KEY>
```
5. Now verify ✅ the system on etherscan with the following command:
```bash
cannon verify kwenta-synthetix-omnibus-test-3:3.3.3-dev.e141cd8c --chain-id 84531 --api-key <ETHERSCAN_API_KEY> 
```
- `<ETHERSCAN_API_KEY>` for everyone on Base Goerli is: `PLACEHOLDER_STRING`
- `kwenta-synthetix-omnibus-test-3:3.3.3-dev.e141cd8c` is comprised of two parts:
  - `kwenta-synthetix-omnibus-test-3` - defined as the name in the cannonfile.
  - `3.3.3-dev.e141cd8c` - the version defined in the cannonfile.
- Awesome, your contracts should be verified now! 🥳 
6. The final step - register your IPFS deployment on the cannon registry! This will allow you to view your contracts on [the online registry](https://usecannon.com/packages). It will show you loads of useful information about your contracts, let you people download the ABIs and contract addresses, and provide a UI for interacting with the contracts (etherscan won't work with the router-proxy-architecture used in synthetix).
   1. To complete this step you need some mainnet ETH. Send some to the EOA on mainnet you want to use for the registry transaction.
   2. Execute the following command:
```bash
cannon publish kwenta-synthetix-omnibus-test-3:3.3.3-dev.e141cd8c --chain-id 84531 --private-key <KEY_THAT_HAS_ETH_ON_MAINNET>
```
- For those interested, this process writes data to the [CannonRegistry contract](https://etherscan.io/address/0xd442Dc2Ac1f3cA1C86C8329246e47Ca0C91D0471#code) on Ethereum mainnet. This is actually just the implementation of contract, it is upgradeable and has a proxy. You can [see the code for the whole system here](https://github.com/usecannon/cannon/tree/main/packages/registry) and specifically [the code for the CannonRegistry contract here](https://github.com/usecannon/cannon/blob/main/packages/registry/contracts/CannonRegistry.sol).

If you made it this far - congratulations! You just deployed synthetix! 🎉

But... we are not finished with smart contract tasks yet from Kwenta's perspective. We still need to do step 7:

7. Deploy `smart-margin-v3` - for that you will need [this repo](https://github.com/Kwenta/smart-margin-v3) - you can use the [Deploy.s.sol script](https://github.com/Kwenta/smart-margin-v3/blob/main/script/Deploy.s.sol) with forge.

## Integrating the front-end the Kwenta deployment

The following steps needed completing once the smart contracts were deployed:
1. Get the ABIs from the [cannon registry site](https://usecannon.com/packages). Got to the deployment page - e.g. [like this one](https://usecannon.com/packages/kwenta-synthetix-omnibus-test-3/3.3.3-dev.e141cd8c/84531-main), scroll down and click the download addresses + ABIs button.
![](./images/download_abis.png)
2. Update the FE with all the new addresses, including:
- Addresses for all synths - e.g. `sUSD`, `sBTC`, `sETH`, `sLINK`, `sOP` etc.
- MarginEngine and depending on the FE setup, the new TrustMulticallForwarder address if applicable.
- A subgraph needs deploying for everything to work fully, however without doing that, the following shortcut can be helpful:
  - Hardcode all settlement strategies to `0`.
- PerpsV3MarketProxy
- AccountProxy
3. Other stuff I am not aware of as a measly smart-contract engineer.

## Interacting with Kwenta Synthetix V3 Base Goerli

Kwenta has now deployed a version of Synthetix V3 on Base Goerli testnet. The name of this deployment is [kwenta-synthetix-omnibus-test-3](https://usecannon.com/packages/kwenta-synthetix-omnibus-test-3/3.3.3-dev.e141cd8c/84531-main). To interact with this can either [use the the UI here](https://usecannon.com/packages/kwenta-synthetix-omnibus-test-3/3.3.3-dev.e141cd8c/84531-main/interact/system/InitialCoreProxy/0x24DE4907c00B0b7aF1E78E1A11a480D047C1b063), or use this command via cannon:
```bash
cannon interact --chain-id 84531 --provider-url <BASE_GOERLI_RPC_URL> --private-key <PRIVATE_KEY> kwenta-synthetix-omnibus-test-3
```
- `84531` is the chain ID for Base Goerli.
- `<BASE_GOERLI_RPC_URL> ` should be an RPC url for an eth node on the Base Goerli network.
- `<PRIVATE_KEY>` can be any private key you want to use when executing transactions against the deployment. Use the Kwenta Testnet Admin (address: 0xC2ecD777d06FFDF8B3179286BEabF52B67E9d991) private key if you want to do privileged actions.
- `kwenta-synthetix-omnibus-test-3` is the name of our testnet deployment.
