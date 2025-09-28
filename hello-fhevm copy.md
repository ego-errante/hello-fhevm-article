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

1. **Install a Node.js TLS version**

   - Ensure that Node.js is installed on your machine.
   - Download and install the recommended LTS (Long-Term Support) version from the [official website](https://nodejs.org/en).
   - Use an even-numbered version (e.g., v18.x, v20.x).
   - To verify your installation:

     ```
     node -v
     npm -v
     ```

2. **Create a new GitHub repository from the FHEVM React template**

   - On GitHub, navigate to the main page of the [FHEVM React template](https://github.com/zama-ai/fhevm-react-template/) repository.
   - Above the file list, click the green Use this template button.
   - Follow the instructions to create a new repository from the FHEVM Hardhat template.

   See Github doc for more info: [Creating a repository from a template](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template#creating-a-repository-from-a-template)

3. **Clone your newly created GitHub repository locally:** Now that your GitHub repository has been created, you can clone it to your local machine:

   ```
   cd <your-preferred-location>
   git clone <url-to-your-new-repo>
   # Navigate to the root of your new FHEVM Hardhat project

   cd <your-new-repo-name>
   ```

4. **Install Dependencies:** Before we install dependencies, let's briefly look at the structure of the project. Here's what the file tree looks like inside your local clone of the repo:

   ```
   <your-new-repo-name>/
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

   We will focus on `packages/fhevm-hardhat-template` for our Hardhat project concerned with writing and deploying our smart contract and `packages/site` for building our interactive frontend with Next.js. To install dependencies for both, we do the following:

   ```bash
   # For the smart contract
   cd packages/fhevm-hardhat-template
   npm install

   # For the frontend
   cd ../site
   npm install
   ```

5. **Set up the Hardhat configuration variables**

   We will be deploying the smart contract to the Sepolia Ethereum Testnet.

   **MNEMONIC**

   A mnemonic is a 12-word seed phrase used to generate your Ethereum wallet keys.

   Get one by creating a wallet with MetaMask, or using any trusted mnemonic generator.

   Set it up in your Hardhat project - `packages/fhevm-hardhat-template`:

   ```bash
   npx hardhat vars set MNEMONIC
   ```

   **INFURA_API_KEY**

   The INFURA project key allows you to connect to Ethereum testnets like Sepolia.

   Obtain one by following the [Infura + MetaMask](https://docs.metamask.io/services/get-started/infura/) setup guide.

   Configure it in your project:

   ```bash
   npx hardhat vars set INFURA_API_KEY
   ```

   **Default Values**

   If you skip this step, Hardhat will fall back to these defaults:

   - MNEMONIC = "test test test test test test test test test test test junk"
   - INFURA_API_KEY = "zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz"

6. **Get Testnet Funds:** You can get Sepolia testnet funds from different providers for free. Here are some faucets:
   - https://faucets.chain.link/sepolia
   - https://www.alchemy.com/faucets/ethereum-sepolia
   - https://cloud.google.com/application/web3/faucet/ethereum/sepolia

## Part 2: The Confidential Smart Contract (`RockPaperScissors.sol`)

In this section, we'll start writing code, starting from our confidential smart contract `RockPaperScissors.sol`.

1.  **The Contract:**

    Below is the complete smart contract code, we'll break it down piece by piece:

    ```
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.8.24;

    import {FHE, euint8, externalEuint8, ebool} from "@fhevm/solidity/lib/FHE.sol";
    import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

    /// @title Rock Paper Scissors Game with FHE
    /// @author Your Name
    /// @notice A privacy-preserving rock paper scissors game using FHEVM
    contract RockPaperScissors is SepoliaConfig {
        /// @notice Game status enumeration
        enum GameStatus {
            Created, // Game created, waiting for second player
            Player1MoveSubmitted, // Player1 submitted move, waiting for player 2
            Resolved // Game has been resolved with encrypted result accessible to players
        }

        /// @notice Game structure
        struct Game {
            address player1;
            address player2;
            euint8 move1; // Encrypted move of player1 (0=rock, 1=paper, 2=scissors)
            euint8 move2; // Encrypted move of player2
            euint8 result; // Encrypted result: 0=draw, 1=player1 wins, 2=player2 wins
            GameStatus status;
            uint256 createdAt;
            uint256 resolvedAt;
        }

        /// @notice Mapping of game ID to game data
        mapping(uint256 => Game) private _games;

        /// @notice Counter for generating unique game IDs
        uint256 private _nextGameId = 1;

        /// @notice Get the next game ID (public getter for latest game ID = nextGameId - 1)
        /// @return The next game ID to be assigned
        function getNextGameId() external view returns (uint256) {
            return _nextGameId;
        }

        /// @notice Events
        event GameCreated(uint256 indexed gameId, address indexed player1);
        event MoveSubmitted(uint256 indexed gameId, address indexed player);
        event GameResolved(uint256 indexed gameId);

        /// @notice Creates a new game
        /// @return gameId The unique identifier for the new game
        function createGame() external returns (uint256 gameId) {
            gameId = _nextGameId++;

            _games[gameId].player1 = msg.sender;
            _games[gameId].player2 = address(0);
            // move1, move2, result remain uninitialized (ZeroHash by default)
            _games[gameId].status = GameStatus.Created;
            _games[gameId].createdAt = block.timestamp;
            _games[gameId].resolvedAt = 0;

            emit GameCreated(gameId, msg.sender);
        }

        /// @notice Gets game information
        /// @param gameId The game identifier
        /// @return player1 Address of player 1
        /// @return player2 Address of player 2
        /// @return move1 Encrypted move of player 1
        /// @return move2 Encrypted move of player 2
        /// @return result Encrypted game result
        /// @return status Current game status
        /// @return createdAt Timestamp when game was created
        /// @return resolvedAt Timestamp when game was resolved
        function getGame(
            uint256 gameId
        )
            external
            view
            returns (
                address player1,
                address player2,
                euint8 move1,
                euint8 move2,
                euint8 result,
                GameStatus status,
                uint256 createdAt,
                uint256 resolvedAt
            )
        {
            Game storage game = _games[gameId];
            require(game.player1 != address(0), "Game does not exist");

            return (
                game.player1,
                game.player2,
                game.move1,
                game.move2,
                game.result,
                game.status,
                game.createdAt,
                game.resolvedAt
            );
        }

        /// @notice Submits an encrypted move for a game
        /// @param gameId The game identifier
        /// @param encryptedMove The encrypted move (0=rock, 1=paper, 2=scissors)
        /// @param inputProof The zero-knowledge proof for the encrypted input
        function submitEncryptedMove(uint256 gameId, externalEuint8 encryptedMove, bytes calldata inputProof) external {
            Game storage game = _games[gameId];
            require(game.player1 != address(0), "Game does not exist");
            require(game.status != GameStatus.Resolved, "Game is already resolved");

            euint8 move = FHE.fromExternal(encryptedMove, inputProof);

            // Validate and normalize move to range (0, 1, 2) using FHE operations
            // This approach clamps any input to the valid range without branching

            // Simple modulo 3 operation using available FHE operations
            // For any input, this will map it to 0, 1, or 2
            euint8 validatedMove = FHE.rem(move, 3);

            if (msg.sender == game.player1) {
                // Player 1 is submitting their move
                require(game.status == GameStatus.Created, "Player1 can only submit in Created state");
                game.move1 = validatedMove;
                game.status = GameStatus.Player1MoveSubmitted;
            } else {
                // This could be player 2 joining and submitting move
                require(game.status == GameStatus.Player1MoveSubmitted, "Player1 must submit their move first");
                if (game.player2 == address(0)) {
                    game.player2 = msg.sender;
                } else {
                    require(msg.sender == game.player2, "Only player1 or player2 can submit moves");
                }
                game.move2 = validatedMove;
                // When player 2 submits their move, we can immediately resolve the game.
                _resolveGame(gameId, game);
            }

            // Allow the contract to use this move in future computations
            // Only allow the submitting player to decrypt their own move
            FHE.allowThis(validatedMove);
            FHE.allow(validatedMove, msg.sender);

            emit MoveSubmitted(gameId, msg.sender);
        }

        /// @notice Internal function to resolve the game and compute the encrypted result
        /// @param gameId The game identifier
        /// @param game The game storage reference
        function _resolveGame(uint256 gameId, Game storage game) internal {
            // Compute the winner using FHE operations without exposing individual moves
            // Rock=0, Paper=1, Scissors=2
            // Rock beats Scissors, Paper beats Rock, Scissors beats Paper

            // Create constants for comparison
            euint8 zero = FHE.asEuint8(0);
            euint8 one = FHE.asEuint8(1);
            euint8 two = FHE.asEuint8(2);

            // Check for draw (move1 == move2)
            ebool isDraw = FHE.eq(game.move1, game.move2);

            // Check all winning conditions for player1:
            // Player1 wins if: (move1=0 && move2=2) || (move1=1 && move2=0) || (move1=2 && move2=1)

            // Player1 plays Rock (0) and Player2 plays Scissors (2)
            ebool p1RockVsP2Scissors = FHE.and(FHE.eq(game.move1, zero), FHE.eq(game.move2, two));

            // Player1 plays Paper (1) and Player2 plays Rock (0)
            ebool p1PaperVsP2Rock = FHE.and(FHE.eq(game.move1, one), FHE.eq(game.move2, zero));

            // Player1 plays Scissors (2) and Player2 plays Paper (1)
            ebool p1ScissorsVsP2Paper = FHE.and(FHE.eq(game.move1, two), FHE.eq(game.move2, one));

            // Player1 wins if any of the above conditions are true
            ebool player1Wins = FHE.or(FHE.or(p1RockVsP2Scissors, p1PaperVsP2Rock), p1ScissorsVsP2Paper);

            // Compute the final result:
            // If draw: result = 0
            // If player1 wins: result = 1
            // If player2 wins: result = 2

            // First, determine if it's a draw (0) or not
            euint8 resultIfNotDraw = FHE.select(player1Wins, one, two);
            game.result = FHE.select(isDraw, zero, resultIfNotDraw);

            // Grant both players access to decrypt ONLY the result, not individual moves
            FHE.allowThis(game.result);
            FHE.allow(game.result, game.player1);
            FHE.allow(game.result, game.player2);

            // DO NOT allow access to individual moves - this preserves privacy
            // Only the contract can access the moves for computation
            FHE.allowThis(game.move1);
            FHE.allowThis(game.move2);

            game.status = GameStatus.Resolved;
            game.resolvedAt = block.timestamp;

            emit GameResolved(gameId);
        }


    }
    ```

    - Our smart contract will handle running game logic (Rock-Paper-Scissors) on encrypted data. Users will submit encrypted data and the contract will compute the winner using FHE.
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

## Others

- gotchas
- tests
- problems
- extending functionality

## References

1. [Zama FHEVM Whitepaper](https://github.com/zama-ai/fhevm/blob/main/fhevm-whitepaper.pdf)
2. [Zama Protocol Litepaper](https://docs.zama.ai/protocol/zama-protocol-litepaper#technical-details)
3. [Zama Setup](https://docs.zama.ai/protocol/solidity-guides/getting-started/setup)
4. [Rock-paper-scissors - Wikipedia](https://en.wikipedia.org/wiki/Rock_paper_scissors)

```

```
