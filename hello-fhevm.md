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

In this section, we'll start writing code, starting from our confidential smart contract `RockPaperScissors.sol`. Our smart contract will handle running game logic (Rock-Paper-Scissors) on encrypted data. Users submit encrypted data and the contract computes the winner using FHE.

1. **Imports and contract declaration**

We need some help from the fhevm solidity library, let's import them. At the top of the file, add:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint8, externalEuint8, ebool} from "@fhevm/solidity/lib/FHE.sol";
import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

/// @title Rock Paper Scissors Game with FHE
/// @author Your Name
/// @notice A privacy-preserving rock paper scissors game using FHEVM
contract RockPaperScissors is SepoliaConfig {

}
```

These imports:

- `FHE` — the core library to work with FHEVM encrypted types
- `euint8` and `externalEuint8` — encrypted `uint8` types used in FHEVM (more on this later). See [list of supported types](https://docs.zama.ai/protocol/solidity-guides/smart-contract/types).
- `SepoliaConfig` — to enable FHEVM support, the contract must inherit from the abstract `SepoliaConfig` contract. Without it, the contract will not be able to execute any FHEVM-related functionality on Sepolia or Hardhat.

2. **Defining the Game: State Variables, Encrypted Types and Events**

Add these variables and types in the contract body. They will control game state:

```solidity
...
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
    mapping(uint256 gameId => Game game) private _games;

    /// @notice Counter for generating unique game IDs
    uint256 private _nextGameId = 1;

    /// @notice Events
    event GameCreated(uint256 indexed gameId, address indexed player1);
    event MoveSubmitted(uint256 indexed gameId, address indexed player);
    event GameResolved(uint256 indexed gameId);
}
...
```

Let take them one at a time:

**State variables**

- `GameStatus` - this enum represents the three different states in the game:
  - `Created` - this is the initial state of a newly created game.
  - `Player1MoveSubmitted` - player 1 has to submit a move before player 2. The game moves to this state when player 1 submits a move.
  - `Resolved` - once player 2 submits their move, the game will automatically resolve, this is the third and final state of the game.
- `Game` - this struct holds the variables of an individual game: players, moves, results etc. Note the use of `euint8` to store moves and result. This is the one of the key elements keeping the game private.
- `_games` - this holds all games in the smart contract. Whenever a game is created, it is added to this list.
- `_nextGameId` - is a simple id generation mechanism. It generates incremental ids for new games.

**Events**

We'll emit events at key points of a game's lifecycle. This allows us to update the frontend in response to those events. If we wanted to make the frontend responsive without these events, the frontend would have to poll the network regularly e.g every 3 seconds. This is a much cleaner approach.

- `GameCreated` - when a new game is created.
- `MoveSubmitted` - when either player submits a move.
- `GameResolved` - when a game completes.

3. **Starting a game: `createGame()`**

```solidity
...
    event MoveSubmitted(uint256 indexed gameId, address indexed player);
    event GameResolved(uint256 indexed gameId);

    /// @notice Creates a new game
    /// @return gameId The unique identifier for the new game
    function createGame() external returns (uint256 gameId) {
        gameId = _nextGameId++;

        _games[gameId] = Game({
            player1: msg.sender,
            player2: address(0),
            move1: FHE.asEuint8(0),
            move2: FHE.asEuint8(0),
            result: FHE.asEuint8(0),
            status: GameStatus.Created,
            createdAt: block.timestamp,
            resolvedAt: 0
        });

        emit GameCreated(gameId, msg.sender);
    }
}
```

The `createGame()` function is our starting point. It handles the standard logic for initializing a game: incrementing a counter for a new `gameId`, assigning the creator as `player1`, setting the status to `Created`, setting default values for for moves and results, and emitting the `GameCreated` event.

4. **Making a Private Move: `submitEncryptedMove()`**

```solidity
        _games[gameId].resolvedAt = 0;

        emit GameCreated(gameId, msg.sender);
    }


    /// @notice Submits an encrypted move for a game
    /// @param gameId The game identifier
    /// @param encryptedMove The encrypted move (0=rock, 1=paper, 2=scissors)
    /// @param inputProof The zero-knowledge proof for the encrypted input
    function submitEncryptedMove(uint256 gameId, externalEuint8 encryptedMove, bytes calldata inputProof) external {
        Game storage game = _games[gameId];
        require(game.player1 != address(0), "Game does not exist");
        require(game.status != GameStatus.Resolved, "Game is already resolved");

        if (msg.sender != game.player1) {
            require(game.status == GameStatus.Player1MoveSubmitted, "Player1 must submit their move first");
        }

        euint8 move = FHE.fromExternal(encryptedMove, inputProof);

        // Validate and normalize move to range (0, 1, 2) using modulo 3 operation in FHE
        // For any input, this will map it to 0, 1, or 2
        euint8 validatedMove = FHE.rem(move, 3);

        if (msg.sender == game.player1) {
            // Player 1 is submitting their move
            require(game.status == GameStatus.Created, "Player1 can only submit in Created state");

            game.move1 = validatedMove;
            game.status = GameStatus.Player1MoveSubmitted;
        } else {
            // This is player 2 joining and submitting their move
            require(game.status == GameStatus.Player1MoveSubmitted, "Player1 must submit their move first");

            game.player2 = msg.sender;
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
}
```

Users submit their moves using the `submitEncryptedMoves` function. Let's go through it step by step:

**Parameters**

- `gameId` - the game the move is being submitted for.
- `encryptedMove` - moves are represented using integer values (0=rock, 1=paper, 2=scissors) but the frontend submits those values using the `externalEuint8` type. `externalEuint8` is an encrypted integer produced off-chain by the function caller (player) and sent to the smart contract.
- `inputProof` - when using `externalEuint8` in a smart contract, we must include this additional argument to ensure the validity of this encrypted value (`encryptedMove`), it is a bytes array containing a Zero-Knowledge Proof of Knowledge (ZKPoK) that proves two things:

  1. The `externalEuint8` was encrypted off-chain by the function caller (msg.sender).
  2. The `externalEuint8` is bound to the contract (`address(this)`) and can only be processed by it.

  These two checks are critical to maintaining the integrity and security of external encrypted data used in the FHEVM, preventing malicious actors from submitting invalid or tampered encrypted values. Conveniently, the `@fhevm/react` library generates this for us on the client-side.

**Logic flow**

The function's core logic acts as a state machine, routing a player's move based on the current status of the game. It uses a sequence of `require` statements and an `if/else` block to manage the game's lifecycle.

Here is the step-by-step flow:

1.  **State Validation:** First, a series of `require` checks ensure the integrity of the game. It confirms the `gameId` is valid, the game hasn't already been resolved, and that the player making the move is authorized to do so at the current stage.

2.  **Move Validation:** Next, move is loaded into the contract and validated.

    - You cannot directly use `externalEuint8` in FHE operations. To manipulate it with the FHEVM library, you first need to convert it into the native FHE type `euint8`.
      This conversion is done using:

      ```solidity
      euint8 move = FHE.fromExternal(encryptedMove, inputProof);
      ```

      This method verifies the zero-knowledge proof and returns a usable encrypted value within the contract.

    - Next, we need to ensure the player's move is valid (i.e., 0 for Rock, 1 for Paper, or 2 for Scissors). In standard Solidity, you might use a `require` statement like `require(move < 3, "Invalid move")`. However, with FHE, we cannot perform conditional checks like `if` or `require` on encrypted values. The result of a comparison like `FHE.lt(move, 3)` is an _encrypted boolean_ (`ebool`), not a clear `true` or `false` that the contract can use to branch or revert.

      Instead, for this use case, we can use a simple and efficient FHE pattern to sanitize the input. By using the FHE remainder operator, we guarantee that any input value is mapped to the valid range `[0, 2]`:

      ```solidity
      // Validate and normalize move to range (0, 1, 2) using modulo 3 operation in FHE
      // For any input, this will map it to 0, 1, or 2
      euint8 validatedMove = FHE.rem(move, 3);
      ```

      This single operation elegantly handles validation without branching or revealing any information about the move itself.

3.  **Move Routing and State Update**: After validation, the contract routes the move and updates game state accordingly:

    - **If `msg.sender` is `player1`** - it verifies that the player hasn't submitted a move by testing for `Created` status, records their move and updates game status.
    - **If `msg.sender` is `player2`** - it verifies that `player1` has submitted their move already by testing for `Player1MoveSubmitted` status, sets `player2` address for the game and records their move. Finally it resolves the game and determines the result with `_resolveGame`.
    - For both `player1` and `player2`, we emit the `MoveSubmitted` event and do some FHE things before ending the function call. Let's explain what's happening there:

      ```solidity
      // Allow the contract to use this move in future computations
      // Only allow the submitting player to decrypt their own move
      FHE.allowThis(validatedMove);
      FHE.allow(validatedMove, msg.sender);
      ```

      Both of these function calls are _required_ to ensure the game works correctly. They allow future smart contract computation on `validatedMove` and enable the caller to decrypt their move off-chain.

      - `FHE.allowThis(validatedMove)` - grants the `RockPaperScissors` contract permission to use `validatedMove` in future transactions.
      - `FHE.allow(validatedMove, msg.sender)` - grants `msg.sender` permission to use `validatedMove`.

      For any encrypted variable either `encryptedMove` or `validatedMove`, if these two permissions are missing, the caller will be unable to decrypt them off-chain.

**5. The Blind Refree: `_resolveGame()`**

    ```solidity

...
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

        game.status = GameStatus.Resolved;
        game.resolvedAt = block.timestamp;

        emit GameResolved(gameId);
    }

}

```

This private function is where the result of the game is determined. It is called internally by `submitEncryptedMove` when player2 submits their move and resolved in that same transaction. It acts as "blind refree", it computes the reult of the game without knowing what the result is or who played what. Let's explore how it does this:

**Logic flow**

The function executes the classic Rock-Paper-Scissors rules, but every operation is performed on _encrypted_ data.

1.  **Checking for a Draw**: The first step is to determine if the game is a draw. In standard Solidity, you'd write `game.move1 == game.move2`. The FHEVM equivalent is `FHE.eq(game.move1, game.move2)`. This function takes two encrypted values as input and returns an `ebool` (an encrypted boolean) which is `true` if they are equal and `false` otherwise. The contract itself cannot see this result, but it can use the `ebool` in subsequent FHE operations.

2.  **Determining the Winner**: Next, we evaluate the win conditions for Player 1. The logic is: Player 1 wins if they play Rock (0) and Player 2 plays Scissors (2), OR Paper (1) vs. Rock (0), OR Scissors (2) vs. Paper (1).

    - We build each individual condition using `FHE.and` and `FHE.eq`. For example, `FHE.and(FHE.eq(game.move1, 0), FHE.eq(game.move2, 2))` checks if Player 1 played Rock and Player 2 played Scissors.
    - We then combine these three winning conditions using `FHE.or`. The final `player1Wins` is an `ebool` that is `true` if any of the winning conditions are met.

3.  **Selecting the Result**: Now that we have encrypted booleans representing `isDraw` and `player1Wins`, we can determine the final result. For this, we use `FHE.select`. This is the FHEVM equivalent of a ternary operator (`condition ? value_if_true : value_if_false`).

    - First, we decide the outcome if it's _not_ a draw: `euint8 resultIfNotDraw = FHE.select(player1Wins, 1, 2)`. This line says, "If `player1Wins` is true, the result is 1; otherwise, the result is 2 (meaning Player 2 wins)."
    - Then, we determine the final result by accounting for the draw condition: `game.result = FHE.select(isDraw, 0, resultIfNotDraw)`. This says, "If `isDraw` is true, the final result is 0; otherwise, use the `resultIfNotDraw` we just calculated."

4.  **Granting Decryption Permission**: At the end, the game is resolved and both players are granted permission to view `game.result`.

    > Notice that we never grant a player the permission to see the other player's move. Although a player can deduce the other player's moves based on the result, it's only because this is a simple game. A player only ever sees their move and the result. Neither the smart contract nor non-participants ever see any player moves or the result of a game.

**6. State getters: `getNextGameId()` and `getGame()`**

```

...
event GameResolved(uint256 indexed gameId);

    /// @notice Get the next game ID (public getter for latest game ID = nextGameId - 1)
    /// @return The next game ID to be assigned
    function getNextGameId() external view returns (uint256) {
        return _nextGameId;
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

    /// @notice Creates a new game
    /// @return gameId The unique identifier for the new game
    function createGame() external returns (uint256 gameId) {

...

```

We expose public getter functions - `getGame()` to fetch game state for a specific game and `getNextGameId()` to fetch the current value of `_nextGameId`, letting the frontend get the most recent game easily.

This completes our confidential contract for the game - players submit their moves privately, our smart contract computes the result confidentially and exposes it securely.

### Part 3: Building the Frontend (React + `@fhevm/react`)

Our confidential smart contract has been created. But how do players interact with it? In this section, we'll build the client-side application that integrates with our confidential smart contract, from creating and submitting encrypted values for computation to decrypting the result for users.

We'll use the `@fhevm/react` library, a powerful toolkit that simplifies all the complex cryptography into a few simple React hooks.

_**A Quick Note on Architecture:** The template's frontend is built with modularity in mind, using a series of custom hooks (`useFhevm`, `useGameActions`, etc.) that work together. While this is great for production code, our focus will be on the **specific FHEVM function calls** within these hooks. We'll follow the user's journey through the dApp to see how it all connects._

For this section, we'll be working in `packages/site`. Keep that in mind.

**1. Add third party libraries**

```

// in packages/site
npm install @tanstack/react-query react-icons

````

This installs `@tanstack/react-query` for clean request management and `react-icons` to provide visual icons for our game interface.

**2. Update `@packages/site/providers.tsx`**

Update your `@packages/site/providers.tsx` to the below:

```typescript
"use client";

import type { ReactNode } from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

import { MetaMaskProvider } from "@/hooks/metamask/useMetaMaskProvider";
import { InMemoryStorageProvider } from "@/hooks/useInMemoryStorage";
import { MetaMaskEthersSignerProvider } from "@/hooks/metamask/useMetaMaskEthersSigner";

// Create a client
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      refetchOnWindowFocus: false,
    },
  },
});

type Props = {
  children: ReactNode;
};

export function Providers({ children }: Props) {
  return (
    <QueryClientProvider client={queryClient}>
      <MetaMaskProvider>
        <MetaMaskEthersSignerProvider
          initialMockChains={{
            11155111: "https://sepolia.infura.io/v3/<your-infura-api-key>",
          }}
        >
          <InMemoryStorageProvider>{children}</InMemoryStorageProvider>
        </MetaMaskEthersSignerProvider>
      </MetaMaskProvider>
    </QueryClientProvider>
  );
}
````

This:

- sets up `react-query` query provider for the frontend.
- configures the default network to ethereum sepolia with a custom rpc url. You can get the `your-infura-api-key` from the infura account get your the rpc url is a using the api you got from infura/metamask dashboard you created in in part 1.

**3. The Core of Frontend Functionality `useRockPaperScissors.tsx`**

Now that our application is set up with providers, let's look at the heart of our game's frontend logic: the `useRockPaperScissors.tsx` hook.

```typescript:hello-fhevm-client/packages/site/hooks/useRockPaperScissors/useRockPaperScissors.tsx
// This file contains the main orchestrating hook that combines all game functionality.
```

Before we explore its responsibilities, let's understand the tools it's given to work with. The `useRockPaperScissors` hook receives several parameters when it's called, but two are essential for all FHEVM operations:

- **`instance: FhevmInstance`**: This is the core object provided by the `@fhevm/react` library. Think of it as our client-side cryptographic engine. It holds the public key needed for encryption and provides the methods we'll use later, such as `encrypt_uint8()` and `decrypt()`. All FHE magic on the frontend happens through this instance.
- **`fhevmDecryptionSignatureStorage: GenericStringStorage`**: This is a helper for caching. To decrypt a value, a user must first sign a message to grant permission. This storage object saves those signatures, so the user doesn't have to sign repeatedly to view the same result.

The other parameters (`ethersSigner`, `chainId`, `userAddress`, etc.) are standard objects from web3 libraries that provide context about the user's wallet and network connection.

This hook is the core of our frontend operations. It doesn't implement all the logic but mainly acts as a central **orchestrator**.

Its key responsibilities are:

- **Combining Sub-Hooks**: It imports and uses our three specialized hooks: `useGameState`, `useGameActions`, and `useGameResults`.
- **Listening for On-Chain Events**: It sets up listeners for our smart contract's events (`GameCreated`, `MoveSubmitted`, `GameResolved`). When one of these events is detected, it automatically tells our app to refetch the latest game state, ensuring the UI is always up-to-date and reactive.
- **Providing a Unified API**: It gathers all the state variables and functions from the sub-hooks and exposes them as a single, clean interface.

By structuring our code this way, we achieve a clean separation of concerns. In the next sections, we will dive into each of the specialized hooks it uses to see exactly how we read game state, handle actions, and process results.

**4. Fetching On-Chain State with `useGameState.ts`**

The first specialized hook our orchestrator uses is `useGameState`. Before our app can display anything meaningful, it needs to know the current state of the game on the blockchain. Is there a game running? Is the current user one of the players? This hook is responsible for answering those questions.

```typescript:hello-fhevm-client/packages/site/hooks/useRockPaperScissors/useGameState.ts
// This hook fetches and manages the current state of the latest Rock Paper Scissors game.
```

The core of this hook is a `useQuery` block from TanStack Query that keeps our frontend synchronized with the smart contract. Here's how it works:

1.  **Contract Identification**: The first thing the hook does is look up the correct smart contract address for the user's currently connected `chainId`. It uses a predefined list of deployed addresses to find the right one. It also provides a boolean flag, `isDeployed`, which becomes `true` only if a valid contract address is found on the current network. This prevents the app from trying to interact with a contract that doesn't exist on the chain.
2.  **Finding the Latest Game**: The query function first calls the `getNextGameId()` view function on our smart contract. Since our game IDs are sequential, this tells us the ID of the most recent game (`nextGameId - 1`).
3.  **Fetching Game Data**: With the latest game ID, it then calls the `getGame()` function to retrieve the full `Game` struct for that game. This object contains all the on-chain information: the player addresses, the game status, and the encrypted moves and result.
4.  **Identifying the User's Role**: After fetching the data, the hook compares the connected `userAddress` with the `player1` and `player2` addresses from the game data. This allows it to determine if the user is `PLAYER1`, `PLAYER2`, or has `NO_ROLE` (is a spectator). This `userGameRole` is essential for showing the correct UI elements (e.g., showing "Submit Move" only to an active player).

It's important to note what this hook _doesn't_ do: it does **not** decrypt anything. When it fetches the game data, the `move1`, `move2`, and `result` fields are just opaque strings of ciphertext. The job of this hook is simply to put the "locked boxes" on the table and tell us who they belong to. In the following sections, we'll see how other hooks are responsible for creating and unlocking them.

See [tanstack query docs](https://tanstack.com/query/latest/docs/framework/react/overview) if you want to learn more about `useQuery` or other tanstack query features.

**5. Making a Move: Encrypting and Submitting with `useGameActions.ts`**

Now that our application can read the game's state, we need to give players a way to interact with it. The `useGameActions` hook is responsible for all "write" operations—actions that change the state of our smart contract. This includes creating a new game and, most importantly, submitting a player's move.

```typescript:hello-fhevm-client/packages/site/hooks/useRockPaperScissors/useGameActions.ts
// This hook provides functions for all game actions, such as creating a game or submitting a move.
```

This hook exposes two key mutations, `createGameMutation` and `submitMoveMutation`, which are built using TanStack Query's `useMutation` for robustly handling asynchronous operations, loading states, and errors. Both mutations also feature calls to `setMessage` to set helpful status updates as the mutation proceeds.

The `createGameMutation` is a standard blockchain transaction that calls the `createGame()` function on our smart contract. The more interesting part is the `submitMoveMutation`, which involves our first client-side FHEVM operation:

1.  **Client-Side Encryption**: When a player chooses a move (e.g., "Paper," represented by the number `1`) and clicks "Submit," this mutation is triggered. The very first thing it does is encrypt the move _inside the user's browser_. It uses the `FhevmInstance` we initialized earlier to do this:

    ```typescript
    const encryptedMove = await instance
      .createEncryptedInput(rockPaperScissors.address, ethersSigner.address)
      .add8(move)
      .encrypt();
    ```

    `createEncryptedInput()` creates an encrypted value bound to both the contract and user addresses. This ensures only the signer can use this encrypted value within the specific contract, preventing reuse/misuse by other users or contracts.

    `add8` defines the type to be converted to as `externalEuint8`.

    This code snippet line takes the plaintext number `1` and transforms it into a secure ciphertext. The user's actual move never leaves their machine unencrypted.

2.  **Submitting the Transaction**: The `encryptedMove` object returned by the `.encrypt()` call contains two critical pieces of data: the ciphertext itself (known as `handles`) and the Zero-Knowledge Proof (`inputProof`). The mutation then calls our smart contract's `submitEncryptedMove` function, passing the `gameId`, the ciphertext, and this input proof.

This matches the function signature we defined in Part 2: `submitEncryptedMove(uint256 gameId, externalEuint8 encryptedMove, bytes calldata inputProof)`.

This hook perfectly illustrates the first half of the FHEVM workflow: encrypting data on the client and sending it to a smart contract for private computation.

**6. Revealing the Winner: Decrypting the Result with `useGameResults.ts`**

Our players have made their moves, and the smart contract has confidentially determined the winner. But the result is still an encrypted secret on the blockchain. The `useGameResults` hook is responsible for the final, and most rewarding, step: securely revealing the outcome to the players.

```typescript:hello-fhevm-client/packages/site/hooks/useRockPaperScissors/useGameResults.ts
// This hook handles fetching and decrypting the game results.
```

When a user clicks "View Results," a function within this hook is triggered. It uses a powerful helper from the `@fhevm/react` library called `FhevmDecryptionSignature` to manage the decryption process. This involves two key steps:

1.  **Authorization and Caching**: Before decrypting, we need the user's permission. The hook manages this with a smart pattern:

    - First, it tries to load a previously generated and stored signature using `FhevmDecryptionSignature.loadFromGenericStringStorage`.
    - If no valid, cached signature is found, it calls `FhevmDecryptionSignature.loadOrSign`. This prompts the user with a standard wallet signature request (which costs no gas).
    - This signature is a cryptographically secure message that serves as a **signed authorization**. It proves the user's identity and gives the FHEVM instance explicit permission to decrypt for our specific contract. Crucially, the helper then **caches this authorization** in the browser's storage. This is a great user experience improvement, as it means the user only has to sign once, and subsequent decryptions for this contract will happen automatically without another prompt.

2.  **Client-Side Decryption**: With the `FhevmDecryptionSignature` object loaded (either from cache or a new signature), the hook can now decrypt the result.

    - It fetches the encrypted `game.result` from our contract.
    - It then prepares the data for decryption. A key detail is that the decryption function requires not just the ciphertext (also known as a "handle"), but also the address of the contract that it came from. The hook prepares a list of objects, each containing a `handle` and a `contractAddress`.
    - Finally, it calls `instance.userDecrypt()`, passing this list and all the necessary components from the signature object.
    - This function takes the ciphertext, its associated contract address, and the signed authorization, and returns the plaintext number, revealing the game's outcome.

    Crucially, this `userDecrypt` call will only succeed for the two players involved in the game. This is because in our smart contract, we used only granted them permission with`FHE.allow` . This on-chain rule is enforced by the FHEVM network. If a spectator tries this decryption flow, the call will fail, protecting the privacy of the game's outcome.

## Others

- gotchas
- tests
- problems
- extending functionality
- ensure consistency in header pattern and language

## References

1. [Zama FHEVM Whitepaper](https://github.com/zama-ai/fhevm/blob/main/fhevm-whitepaper.pdf)
2. [Zama Protocol Litepaper](https://docs.zama.ai/protocol/zama-protocol-litepaper#technical-details)
3. [Zama Solidity Guides](https://docs.zama.ai/protocol/solidity-guides/)
4. [Rock-paper-scissors - Wikipedia](https://en.wikipedia.org/wiki/Rock_paper_scissors)
