## Part 4: Running the Full dApp End-to-End

Our dApp is complete from smart contract to frontend. To see it in action, we need deploy it then launch it.

**4.1. Update Deployment Script**

We need to update the deployment script in `packages/fhevm-hardhat-template/deploy/deploy.ts` to this:

```typescript
import { DeployFunction } from "hardhat-deploy/types";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { postDeploy } from "postdeploy";

const func: DeployFunction = async function (hre: HardhatRuntimeEnvironment) {
  const { deployer } = await hre.getNamedAccounts();
  const { deploy } = hre.deployments;

  const chainId = await hre.getChainId();
  const chainName = hre.network.name;

  const contractName = "RockPaperScissors";
  const deployed = await deploy(contractName, {
    from: deployer,
    log: true,
  });

  console.log(`${contractName} contract address: ${deployed.address}`);
  console.log(`${contractName} chainId: ${chainId}`);
  console.log(`${contractName} chainName: ${chainName}`);

  // Generates:
  //  - <root>/packages/site/abi/RockPaperScissorsABI.ts
  //  - <root>/packages/site/abi/RockPaperScissorsAddresses.ts
  postDeploy(chainName, contractName);
};

export default func;

func.id = "deploy_rockPaperScissors"; // id required to prevent reexecution
func.tags = ["RockPaperScissors"];
```

Now it deploys `RockPaperScissors.sol` instead of the old `FHECounter.sol`.

`postDeploy` function defined in `hello-fhevm/packages/postdeploy/index.ts` is a helper function that generates the ABI and address files (`RockPaperScissorsABI.ts` and `RockPaperScissorsAddresses.ts`) used by the frontend. It also puts them in the right location so after deployment, the frontend 'just works'.

**4.2. Deploy the Contract to Ethereum Sepolia Testnet**

All this commands are run from local repo root - `hello-fhevm/` since it is a monorepo.

Run:

```
npm run deploy:sepolia
```

**4.3. Launch the Frontend**

The `fhevm-react-template` is setup to test for a local running hardhat node. Even though we deployed on Sepolia, let's start that first. Run:

```
npm run hardhat-node
```

After giving it some seconds to come up, start the frontend:

```
npm run dev:mock
```

**4.4. Game play**

In the gif below, I use two different browsers to show a game between two imaginary players A and B. Player A (left screen) plays Paper, player B (right screen) plays rock resolving the game. At the end, player B wins.

```gameplay video

```
