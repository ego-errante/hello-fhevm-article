### Part 3: Building the Frontend (React + `@fhevm/react`)

Our confidential smart contract has been created. But how do players interact with it? In this section, we'll build the client-side application that integrates with our confidential smart contract, from creating and submitting encrypted values for computation to decrypting the result for users.

We'll use the `@fhevm/react` library, a powerful toolkit that simplifies all the complex cryptography into a few simple React hooks.

_**A Quick Note on Architecture:** The template's frontend is built with modularity in mind, using a series of custom hooks (`useFhevm`, `useGameActions`, etc.) that work together. While this is great for production code, our focus will be on the **specific FHEVM function calls** within these hooks. We'll follow the user's journey through the dApp to see how it all connects._

For this section, we'll be working in `packages/site`. Keep that in mind.

**1. Add third party libraries**

```

// in packages/site
npm install @tanstack/react-query react-icons

```

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
```

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

**7. Putting the UI together**

We have gone through the hooks that power the frontend of our dApp. They implement the full encryption, submission and decryption cycle. The final step is building the React components that use these hooks to create a functional user interface. We'll follow a top-down approach, starting from the application's root layout to see how everything is connected.

**7.1 The Root Layout (`app/layout.tsx`)**

Starting with our layout file in `packages/site/app/layout.tsx`. Copy this code:

```typescript:hello-fhevm-client/packages/site/app/layout.tsx
// This file sets up the main layout, including wrapping the application with the Providers component.
```

We made a few changes: added a new font (`Funnel_Sans`), and updated the metadata and app logo to better match our dApp.

**7.2. The Home Page (`app/page.tsx`)**

Next, we'll set up the main page of our site, `packages/site/app/page.tsx`. Copy this code

```typescript:hello-fhevm-client/packages/site/app/page.tsx
// This file renders the main page, including the title and the RockPaperScissorsDemo component.
```

We replaced the `FHECounterDemo` with our new `RockPaperScissorsDemo` component.

**7.3. The Main Event: A Guided Build of `RockPaperScissorsDemo.tsx`**

We will now build our primary interactive component, `RockPaperScissorsDemo`, piece by piece.

**7.3.1. Hooks and `RockPaperScissorsDemo` Overview**

Let's create the file `packages/site/components/RockPaperScissorsDemo.tsx` and add this code:

```typescript:hello-fhevm-client/packages/site/components/RockPaperScissorsDemo.tsx
// This file will contain the main component for the Rock Paper Scissors game demo.
```

Inside our component, we first call `useMetaMaskEthersSigner()` to provide wallet functionality, `useInMemoryStorage` to get `fhevmDecryptionSignatureStorage` instance, `useFhevm()` to intialize the FHEVM library, providing the FHEVM instance object we'll use for all client-side FHE cryptography. We then pass all of this information into our main orchestrator hook, `useRockPaperScissors`.

With all this setup, our component now has access to everything it needs to display game state, handle user actions, and show results.

Understood. Apologies, my previous draft did not align with the structure of your article. Let's correct that by continuing from where you left off, exploring the presentational components one by one as requested.

Here is the improved draft.

**7.3.2. Displaying Game State: `GameStatusBoxSection`**

This component is the primary user-facing panel. It is responsible for displaying the current state of the game and presenting the player with the correct action button.

Create and copy code for these files that make up the component:

```typescript:hello-fhevm-client/packages/site/components/GameStatus.tsx
// This file contains the GameStatusBoxSection component, which acts as the main display panel.
```

```typescript:hello-fhevm-client/packages/site/components/GameButton.tsx
// This file defines a reusable button component for all game actions.
```

```typescript:hello-fhevm-client/packages/site/components/GameResult.tsx
// This component is responsible for displaying the final outcome of the game.
```

The `GameStatusBoxSection` conditionally render the appropriate view - `Player1View`, `Player2View` or `SpectatorView` depending on the current user's role in the latest game

**7.3.3. Capturing Player Input: The `MoveSelectorModal`**

When submitting a move or joining a game, the `RockPaperScissorsDemo` component shows the `MoveSelectorModal`. This component's allow a user to select their move - Rock, Paper, or Scissors and submit

```typescript:hello-fhevm-client/packages/site/components/MoveSelector.tsx
// This file contains the modal UI for selecting a move.
```

This component is the final step in the UI before triggering client-side FHEVM encryption and the submitting on-chain transaction.

**7.3.4. Supporting UI Components**

The remaining components in `RockPaperScissorsDemo` play supporting roles in the user experience.

```typescript:hello-fhevm-client/packages/site/components/MessageSection.tsx
// This component displays informational or error messages to the user.
```

```typescript:hello-fhevm-client/packages/site/components/InfoPanels.tsx
// This component (which exports TechnicalDetailsSection) shows debugging info like chain ID and contract address.
```

```typescript:hello-fhevm-client/packages/site/components/ConnectButton.tsx
// This component handles the wallet connection flow.
```

```typescript:hello-fhevm-client/packages/site/components/ErrorNotDeployed.tsx
// This component shows an error if the user is on a network where the contract is not deployed.
```

Here is a breakdown of what they do component does:

- **`ConnectButton`**: This provides the button to initiate a wallet connection, it is rendered if a connection doesn't already exist.
- **`errorNotDeployed`**: If the user connects to a network where the `RockPaperScissors` contract address is not found, this component is rendered to inform them of the issue.
- **`MessageSection`**: Displays dynamic, contextual messages to the user. It shows helpful status updates like "[FHE::Arena] Initializing encrypted battleground..."
- **`TechnicalDetailsSection`**: A collapsible panel for developers or curious users. It shows helpful debugging information like the connected `chainId`, wallet `accounts`, the `contractAddress`, and the `fhevmInstance` status.

These components complete the application's UI. They are not directly involved in the FHEVM encryption or decryption flows, which are all handled by the hooks and orchestrated by the main `RockPaperScissorsDemo` component.

**7.4. Shared Utilities: The `/lib` Directory**

Before we conclude, let's add the shared utilities in `packages/site/lib` directory.

```typescript:hello-fhevm-client/packages/site/lib/types.ts
// This file defines shared TypeScript types used across the application.
```

```typescript:hello-fhevm-client/packages/site/lib/constants.ts
// This file defines shared constants, such as move names and game outcomes.
```
