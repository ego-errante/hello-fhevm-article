# Building Your First Confidential DApp: A "Hello FHEVM" Tutorial with Rock-Paper-Scissors

## Introduction

What if you could build dApps where user data remains private? Multiple users can submit data, smart contracts can compute on that data, all while that data remains hidden from other users and even the smart contract itself. That might sound like magic but that's exactly what Zama's FHEVM enables.

### What is FHEVM?

From the [Zama whitepaper](https://github.com/zama-ai/fhevm/blob/main/fhevm-whitepaper.pdf)

> Zama's fhevm is a cross-chain protocol that enables confidential smart contracts on any L1 and L2 using Fully Homomorphic Encryption (FHE). It brings end-to-end encryption to onchain applications, guaranteeing confidentiality and privacy while remaining fully composable, verifiable, and permissionless

Did you get that? If you didn't, it's alright. Let's break that down:

- FHE is technology that enables computation over encrypted data. Imagine voting in an election where each person's vote is private, but the election result can be computed without revealing any single person's vote. Complex math behind the scenes makes this possible.

- FHEVM is a collection of technologies that work together to bring FHE capabilities to onchain applications on any L1 or L2. Developers get to use familiar languages like Solidity without having to think about the mathematical complexity behind it. Feel free to read the [whitepaper](https://github.com/zama-ai/fhevm/blob/main/fhevm-whitepaper.pdf) or [litepaper](https://docs.zama.ai/protocol/zama-protocol-litepaper#technical-details) for more info.

### What We'll Build

To demonstrate the the power of FHE and the ease of FHEVM, we're going to build a Rock-Paper-Scissors dApp. It's a [classic game](https://en.wikipedia.org/wiki/Rock_paper_scissors), but here's the twist - with the power of FHE, users will submit encrypted moves, the contract determines the winner without ever knowing the moves or the result. Only the players can see the result.

### Learning Objectives

At the end of this tutorial, the reader will hopefully:

- Understand the basics of FHEVM and why it matters.
- Be able to set up the dev environment.
- Deploy and interact with a simple FHEVM dApp end-to-end.
- Be confident to start experimenting with more advanced use cases.

### Prerequisites

To follow along, no prior knowledge of FHE or cryptography is required; the tutorial assumes zero background in advanced math or cryptography. However, the reader is expected to:

- Have basic Solidity knowledge (comfortable writing and deploying simple smart contracts).
- Be familiar with standard Ethereum dev tools (e.g. Hardhat, Foundry, Metamask).
- Have basic familiarity with React.

## Part 1: Setting Up Your FHEVM Environment

To get started, let's setup up our dev environment.

**1.1. Install a Node.js TLS version**

- Ensure that Node.js is installed on your machine.
- Download and install the recommended LTS (Long-Term Support) version from the [official website](https://nodejs.org/en).
- Use an even-numbered version (e.g., v18.x, v20.x).
- To verify your installation:

  ```
  node -v
  npm -v
  ```

**1.2. Create a new GitHub repository from the FHEVM React template**

- On GitHub, navigate to the main page of the [FHEVM React template](https://github.com/zama-ai/fhevm-react-template/) repository.
- Above the file list, click the green Use this template button.
- Follow the instructions to create a new repository from the FHEVM Hardhat template. You can name it anything you like (e.g., "my-fhevm-dapp", "rock-paper-scissors-fhe", etc.).

**Note:** For this tutorial, we'll assume you named your repository `hello-fhevm`. If you chose a different name, simply substitute `hello-fhevm` with your chosen repository name throughout this tutorial.

See Github doc for more info: [Creating a repository from a template](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template#creating-a-repository-from-a-template)

**1.3. Clone your newly created GitHub repository locally:**
Now that your GitHub repository has been created, you can clone it to your local machine:

```
cd <your-preferred-location>
git clone <url-to-your-new-repo>
# Navigate to the root of your new FHEVM Hardhat project

cd hello-fhevm
```

**1.4. Install Dependencies:** Before we install dependencies, let's briefly look at the structure of the project. Here's what the file tree looks like inside your local clone of the repo:

```
hello-fhevm/
├── LICENSE
├── README.md
├── package-lock.json
├── package.json
├── packages/
│   ├── fhevm-hardhat-template/
│   ├── fhevm-react/
│   ├── postdeploy/
│   └── site/
```

We will focus on `hello-fhevm/packages/fhevm-hardhat-template` for our Hardhat project concerned with writing and deploying our smart contract and `hello-fhevm/packages/site` for building our interactive frontend with Next.js.
Before installing, we'll update the package.json files to remove scripts that are only needed for local Hardhat development.

Remove this `postinstall` script line in `hello-fhevm/package.json`:

```json
...
    "postinstall": "./scripts/deploy-hardhat-node.sh",
...
```

In `hello-fhevm/packages/site/package.json`, update the `dev:mock` command by removing the `npm run is-hardhat-node-running` test, leaving just:

```json
...
    "dev:mock": "next dev --turbopack",
...
```

To install dependencies for both, run:

```bash
# in the repo root
npm install
```

**1.5. Set up the Hardhat configuration variables**

We will be deploying the smart contract to the Sepolia Ethereum Testnet.

**1.5.1. MNEMONIC**

A mnemonic is a 12-word seed phrase used to generate your Ethereum wallet keys.

Get one by creating a wallet with MetaMask, or using any trusted mnemonic generator.

Set it up in your Hardhat project - `hello-fhevm/packages/fhevm-hardhat-template`:

```bash
npx hardhat vars set MNEMONIC
```

**1.5.2. INFURA_API_KEY**

The INFURA project key allows you to connect to Ethereum testnets like Sepolia.

Obtain one by following the [Infura + MetaMask](https://docs.metamask.io/services/get-started/infura/) setup guide.

Configure it in your project:

```bash
npx hardhat vars set INFURA_API_KEY
```

**1.5.3. Default Values**

If you skip this step, Hardhat will fall back to these defaults:

- MNEMONIC = "test test test test test test test test test test test junk"
- INFURA_API_KEY = "zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz"

**1.6. Get Testnet Funds:** You can get Sepolia testnet funds from different providers for free. Here are some faucets:

- https://faucets.chain.link/sepolia
- https://www.alchemy.com/faucets/ethereum-sepolia
- https://cloud.google.com/application/web3/faucet/ethereum/sepolia
