# Building Your First Confidential DApp: A "Hello FHEVM" Tutorial with Rock-Paper-Scissors

## Introduction

- **Hook:** Start with a relatable problem. "Ever played a game on the blockchain and wished your moves could be secret? What if you could build dApps where user data remains private, even from the smart contract itself?"
- **What is FHE?** Briefly explain Fully Homomorphic Encryption (FHE) in simple terms. Analogy: "It's like having a locked box where you can perform operations on the contents without ever opening it."
- **Introducing FHEVM:** Explain what FHEVM is (an extension for EVM chains that enables confidential smart contracts) and why it's a game-changer for on-chain privacy.
- **What We'll Build:** Introduce the Rock-Paper-Scissors dApp. It's a fun, simple example that perfectly demonstrates the power of FHE: players submit their moves encrypted, the contract determines the winner without ever knowing the moves, and only the players can see the result.
- **Learning Objectives:**
  - Understand the fundamentals of FHEVM.
  - Set up a complete FHEVM development environment.
  - Build and deploy a confidential smart contract.
  - Create a frontend to interact with the contract, handling encryption and decryption.
- **Prerequisites:**
  - Basic knowledge of Solidity and smart contract development.
  - Familiarity with React (or another frontend framework).
  - Standard Web3 tools installed: Node.js, MetaMask.

---

## Part 1: Setting Up Your FHEVM Environment

1.  **Clone the Project:** Provide the link to your GitHub repository.
    ```bash
    git clone <your-repo-link>
    cd <your-repo-folder>
    ```
2.  **Install Dependencies:** Explain the project structure (`contracts` for the Hardhat project and `hello-fhevm-client/packages/site` for the Next.js frontend) and provide commands to install dependencies for both.

    ```bash
    # For the smart contract
    cd contracts
    npm install

    # For the frontend
    cd ../hello-fhevm-client/packages/site
    npm install
    ```

3.  **Configure MetaMask for the Zama Testnet:**
    - Provide the network details (Network Name, RPC URL, Chain ID, etc.).
    - Link to the official Zama documentation for adding the network.
4.  **Get Testnet Funds:** Link to the official Zama faucet and guide users on how to get tokens for deployment and testing.

---

## Part 2: The Confidential Smart Contract (`RockPaperScissors.sol`)

_In this section, we'll dive into the heart of our dApp: the confidential smart contract that runs the game logic on encrypted data._

1.  **Contract Overview:**

    - Explain the goal: to create a Rock-Paper-Scissors game where player moves are private.
    - Walk through the main components of `RockPaperScissors.sol`:
      - `GameStatus` enum: `Created`, `Player1MoveSubmitted`, `Resolved`.
      - `Game` struct: `player1`, `player2`, `move1`, `move2`, `result`. Highlight that `move1`, `move2`, and `result` are of type `euint8` (encrypted unsigned 8-bit integer).

2.  **Core Functions:**

    - **`createGame()`**: A simple function to initialize a new game, setting `player1` to `msg.sender`.
    - **`submitEncryptedMove(uint256 gameId, externalEuint8 encryptedMove, bytes calldata inputProof)`**: This is where the FHE magic begins.
      - Explain that the `encryptedMove` is not a simple number, but an encrypted value sent from the frontend.
      - Show `euint8 validatedMove = FHE.rem(move, 3);`. This is a crucial FHE operation to ensure the move is valid (0, 1, or 2) without decrypting it.
      - Explain how the contract handles moves from `player1` and `player2` and changes the game status.
    - **`_resolveGame(uint256 gameId, Game storage game)`**: The core of the confidential computation.
      - **Emphasize:** This entire function operates on _encrypted data_. The contract is a "blind" referee.
      - Show the FHE logic for determining the winner:
        - `ebool isDraw = FHE.eq(game.move1, game.move2);` (checking for equality on encrypted values).
        - `ebool player1Wins = FHE.or(FHE.or(p1RockVsP2Scissors, p1PaperVsP2Rock), p1ScissorsVsP2Paper);` (using `FHE.and` and `FHE.or` to check winning conditions).
        - `game.result = FHE.select(isDraw, zero, resultIfNotDraw);` (selecting the result based on conditions, all with encrypted data).

3.  **Privacy and Access Control:**
    - Explain the importance of `FHE.allow()`.
    - Show how we grant decryption permission for the result to both players, but NOT for each other's moves. This enforces the privacy of the game.
    ```solidity
    // Grant players access to decrypt ONLY the result
    FHE.allow(game.result, game.player1);
    FHE.allow(game.result, game.player2);
    ```

---

## Part 3: Building the Frontend (React + `fhevmjs`)

_Now, let's build the user interface that will allow players to interact with our confidential smart contract. We'll use the `fhevmjs` library to handle the client-side encryption and decryption._

1.  **Introducing `fhevmjs`:**

    - Explain that `fhevmjs` is the bridge between the frontend and FHEVM. It provides tools to:
      - Connect to a user's wallet and the FHEVM provider.
      - Encrypt data on the client-side before sending it to the contract.
      - Decrypt data received from the contract.

2.  **The Complete Workflow: Encryption → Computation → Decryption**

    - **Step 1: Initialization.** Show how to create an instance of the FHEVM provider in your React hooks (e.g., in `hooks/useFhevm.tsx`).
    - **Step 2: Encryption.**
      - Before a player can submit a move, the frontend needs to encrypt it.
      - Show the code snippet for encrypting the player's chosen move (0 for Rock, 1 for Paper, 2 for Scissors) using `fhevmjs`. Explain that this happens entirely in the user's browser.
    - **Step 3: On-Chain Computation.**
      - Show how to call the `submitEncryptedMove` function from the frontend, passing the encrypted move.
      - Reiterate that the contract now performs the game logic on this encrypted data.
    - **Step 4: Decryption.**
      - After the game is resolved, show how the frontend fetches the encrypted `result` from the contract.
      - Display the `fhevmjs` code that decrypts the result. Explain that only `player1` and `player2` (who were granted permission) can successfully decrypt this value.

3.  **Key Frontend Components (from `packages/site`):**
    - Briefly explain the role of the main components and hooks:
      - `hooks/useRockPaperScissors/`: The main logic for game state, actions, and results.
      - `components/RockPaperScissorsDemo.tsx`: The main component that ties everything together.
      - `components/GameStatus.tsx`: To display the current state of the game.

---

## Part 4: Running the Full dApp End-to-End

1.  **Deploy the Contract:**

    - Guide the user to the deploy script in the `contracts/deploy` folder.
    - Provide the command to deploy the `RockPaperScissors` contract to the Zama testnet.

    ```bash
    npx hardhat deploy --network zama-testnet
    ```

    - Instruct them to copy the deployed contract address.

2.  **Configure the Frontend:**

    - Show where to paste the contract address in the frontend configuration files.

3.  **Launch and Play:**
    - Provide the command to start the frontend application.
    ```bash
    npm run dev
    ```
    - Provide a step-by-step visual walkthrough (using screenshots or a GIF) of a full game:
      1. Player 1 connects their wallet and creates a new game.
      2. Player 1 encrypts and submits their move (e.g., "Rock").
      3. Player 2 (using a different browser/wallet) connects and joins the game.
      4. Player 2 encrypts and submits their move (e.g., "Paper").
      5. The frontend automatically fetches and decrypts the result, declaring Player 2 as the winner.
      6. Highlight that at no point were the moves revealed to the public or each other.

---

## Conclusion

- **Recap:** Briefly summarize what the developer has accomplished: setting up an FHEVM environment, deploying a confidential smart contract, and building a frontend to interact with it using client-side encryption/decryption.
- **You've Built Your First FHEVM DApp!** Congratulate the reader.
- **What's Next?**
  - Encourage them to experiment. "Try adding a new feature, like a betting mechanism!"
  - Point them to the official Zama documentation, the FHEVM SDK, and the Zama Discord for more advanced use cases.
- **Final Links:**
  - Link to the completed GitHub repository.
  - (Optional) Link to a live deployed demo on the Zama Testnet.
