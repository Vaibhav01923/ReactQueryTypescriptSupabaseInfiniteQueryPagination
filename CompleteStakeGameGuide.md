# The Ultimate Stake Engine Game Development Guide

**Version 1.0**

This guide is a complete, standalone manual for building a frontend game compatible with the Stake Engine RGS (Remote Game Server). It is designed so that you can hand this document to an AI or a developer, and they can build a fully functional game without needing extra context.

---

## Table of Contents

1.  [Project Initialization](#1-project-initialization)
2.  [Directory Structure](#2-directory-structure)
3.  [Core Types & Interfaces](#3-core-types--interfaces)
4.  [The RGS API Layer](#4-the-rgs-api-layer)
5.  [State Management (Game Context)](#5-state-management-game-context)
6.  [Game Logic & Flow](#6-game-logic--flow)
7.  [Mocking for Local Development](#7-mocking-for-local-development)
8.  [Asset Management (2D & 3D)](#8-asset-management-2d--3d)
9.  [Building for Production](#9-building-for-production)

---

## 1. Project Initialization

We use **Vite + React + TypeScript** for the best balance of performance and developer experience.

```bash
# 1. Create the project
npm create vite@latest my-stake-game -- --template react-ts

# 2. Enter directory
cd my-stake-game

# 3. Install dependencies
npm install
npm install three @types/three @react-three/fiber @react-three/drei # If using 3D
npm install clsx tailwind-merge # Recommended for styling
```

---

## 2. Directory Structure

Organize your project exactly like this to ensure scalability.

```text
/public
  /models          <-- GLTF/GLB files go here (NOT in src)
    chest.glb
  /sounds          <-- Audio files
/src
  /assets          <-- 2D images (sprites, UI icons)
  /components      <-- React UI components
    /Game
      Board.tsx
      Controls.tsx
    /UI
      Header.tsx
      BalanceDisplay.tsx
  /context         <-- Global State
    GameContext.tsx
  /lib             <-- Core Logic
    rgs-api.ts     <-- The Bridge to Stake Engine
    mock-rgs.ts    <-- For local testing
    types.ts       <-- TypeScript definitions
  App.tsx
  main.tsx
```

---

## 3. Core Types & Interfaces

Define the shape of the data coming from the RGS. This is crucial. Based on standard Stake Engine patterns:

**`src/lib/types.ts`**

```typescript
// The standard response from any RGS call
export interface RGSResponse {
  balance: {
    amount: number; // e.g., 1000000000 (integers, 1.00 = 1,000,000)
    currency: string; // "USD", "EUR", etc.
  };
  round: GameRound | null;
  config?: GameConfig; // Only present in authenticate
}

export interface GameRound {
  betID: number;
  amount: number;
  payout: number;
  payoutMultiplier: number;
  active: boolean; // TRUE = Game in progress, FALSE = Game Over/Idle
  state: GameState[]; // Array of state updates
  mode: string; // "CLASSIC", "BONUS", etc.
}

// Customize this based on YOUR specific game's JSON
export interface GameState {
  index: number;
  type: string; // e.g., "MINES_CLONE_ROUND"
  board?: string[]; // e.g., ["hero_1", "hero_2", ...]
  winning_hero?: string;
  totalWin?: number;
  // ... add other fields from your specific RGS response
}

export interface GameConfig {
  minBet: number;
  maxBet: number;
  betLevels: number[];
  jurisdiction: {
    socialCasino: boolean;
    disabledAutoplay: boolean;
    // ...
  };
}
```

---

## 4. The RGS API Layer

This class handles all communication. Copy this file exactly.

**`src/lib/rgs-api.ts`**

```typescript
import { RGSResponse } from "./types";

export class RGSAPI {
  private baseUrl: string;

  constructor(baseUrl: string) {
    // Ensure no trailing slash
    this.baseUrl = baseUrl.replace(/\/$/, "");
  }

  private async request<T>(endpoint: string, body: any): Promise<T> {
    try {
      const response = await fetch(`${this.baseUrl}/wallet/${endpoint}`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(body),
      });

      if (!response.ok) {
        const error = await response.json().catch(() => ({}));
        throw new Error(error.message || `API Error: ${response.status}`);
      }

      return await response.json();
    } catch (err) {
      console.error(`RGS Request Failed [${endpoint}]:`, err);
      throw err;
    }
  }

  async authenticate(
    sessionID: string,
    isSocial: boolean = false
  ): Promise<RGSResponse> {
    return this.request<RGSResponse>("authenticate", {
      sessionID,
      socialCasino: isSocial,
    });
  }

  async play(
    sessionID: string,
    amount: number,
    mode: string,
    payload: any = {}
  ): Promise<RGSResponse> {
    return this.request<RGSResponse>("play", {
      sessionID,
      amount,
      mode: mode.toUpperCase(),
      ...payload, // Game specific data (e.g. tileIndex, minesCount)
    });
  }

  async endRound(sessionID: string): Promise<RGSResponse> {
    return this.request<RGSResponse>("end-round", { sessionID });
  }
}
```

---

## 5. State Management (Game Context)

Use a React Context to make game state available everywhere.

**`src/context/GameContext.tsx`**

```typescript
import React, { createContext, useContext, useState, useEffect } from "react";
import { RGSAPI } from "../lib/rgs-api";
import { MockRGS } from "../lib/mock-rgs"; // See Section 7
import { RGSResponse, GameRound, GameConfig } from "../lib/types";

interface GameContextType {
  isLoading: boolean;
  balance: number;
  round: GameRound | null;
  config: GameConfig | null;
  play: (amount: number, mode: string, payload?: any) => Promise<void>;
}

const GameContext = createContext<GameContextType | null>(null);

export const GameProvider: React.FC<{ children: React.ReactNode }> = ({
  children,
}) => {
  const [api, setApi] = useState<RGSAPI | MockRGS | null>(null);
  const [sessionID, setSessionID] = useState<string>("");

  const [isLoading, setIsLoading] = useState(true);
  const [balance, setBalance] = useState(0);
  const [round, setRound] = useState<GameRound | null>(null);
  const [config, setConfig] = useState<GameConfig | null>(null);

  // 1. Initialize on Mount
  useEffect(() => {
    const init = async () => {
      const params = new URLSearchParams(window.location.search);
      const sid = params.get("sessionID");
      const url = params.get("rgs_url");
      const isSocial = params.get("social") === "true";
      const isDev = !sid || !url; // If no params, use Mock

      let rgs: RGSAPI | MockRGS;
      let currentSession = sid || "test-session";

      if (isDev) {
        console.warn("Using Mock RGS");
        rgs = new MockRGS();
      } else {
        rgs = new RGSAPI(url!);
      }

      setApi(rgs);
      setSessionID(currentSession);

      try {
        const data = await rgs.authenticate(currentSession, isSocial);
        setBalance(data.balance.amount);
        setConfig(data.config || null);
        setRound(data.round);
      } catch (e) {
        console.error("Failed to init game", e);
      } finally {
        setIsLoading(false);
      }
    };

    init();
  }, []);

  // 2. The Play Action
  const play = async (amount: number, mode: string, payload?: any) => {
    if (!api || !sessionID) return;

    try {
      const data = await api.play(sessionID, amount, mode, payload);

      // Update Global State
      setBalance(data.balance.amount);
      setRound(data.round);

      // Handle Game Over / Win logic here or in components
      if (data.round?.active === false) {
        console.log("Game Over! Win:", data.round.payout);
      }
    } catch (e) {
      console.error("Play error", e);
    }
  };

  return (
    <GameContext.Provider value={{ isLoading, balance, round, config, play }}>
      {children}
    </GameContext.Provider>
  );
};

export const useGame = () => {
  const context = useContext(GameContext);
  if (!context) throw new Error("useGame must be used within GameProvider");
  return context;
};
```

---

## 6. Game Logic & Flow

Your `App.tsx` should look like this:

```typescript
import { GameProvider, useGame } from "./context/GameContext";
import Header from "./components/UI/Header";
import GameBoard from "./components/Game/GameBoard";
import Controls from "./components/Game/Controls";

const GameContent = () => {
  const { isLoading, round } = useGame();

  if (isLoading) return <div>Loading Game...</div>;

  return (
    <div className="game-container">
      <Header />
      <div className="main-area">
        <GameBoard round={round} />
      </div>
      <Controls />
    </div>
  );
};

export default function App() {
  return (
    <GameProvider>
      <GameContent />
    </GameProvider>
  );
}
```

### Mapping Response to Visuals

In `GameBoard.tsx`, you map the `round.state` to your UI.

```typescript
// Example for Mines/Grid game
const GameBoard = ({ round }: { round: GameRound | null }) => {
  // If no round, show empty board
  if (!round) return <EmptyBoard />;

  // Get latest state
  const currentState = round.state[round.state.length - 1];
  const boardData = currentState.board || []; // ["hero_1", "hero_2", ...]

  return (
    <div className="grid">
      {boardData.map((cell, index) => (
        <Cell key={index} type={cell} />
      ))}
    </div>
  );
};
```

---

## 7. Mocking for Local Development

Since you don't have the RGS running locally, you **MUST** mock it. This allows you to build the UI using the exact JSON structure you expect.

**`src/lib/mock-rgs.ts`**

```typescript
import { RGSResponse } from "./types";

export class MockRGS {
  async authenticate(
    sessionID: string,
    isSocial: boolean
  ): Promise<RGSResponse> {
    // Return your sample AUTHENTICATE JSON here
    return {
      balance: { amount: 1000000000, currency: "USD" },
      round: null, // Or a sample active round
      config: {
        minBet: 100000,
        maxBet: 1000000000,
        betLevels: [100000, 200000, 500000],
        jurisdiction: { socialCasino: false, disabledAutoplay: false },
      },
    } as any;
  }

  async play(
    sessionID: string,
    amount: number,
    mode: string,
    payload: any
  ): Promise<RGSResponse> {
    console.log("MOCK PLAY:", { amount, mode, payload });

    // Simulate network delay
    await new Promise((r) => setTimeout(r, 500));

    // Return your sample PLAY JSON here
    return {
      balance: { amount: 999000000, currency: "USD" },
      round: {
        betID: 123456,
        amount: amount,
        payout: 0,
        payoutMultiplier: 0,
        active: true,
        state: [
          {
            index: 0,
            type: "MOCK_ROUND",
            board: ["hero_1", "hero_1", "hero_2"], // Simulate a board update
          },
        ],
        mode: mode,
      },
    } as any;
  }
}
```

---

## 8. Asset Management (2D & 3D)

### 2D Images

Put them in `src/assets`.

```typescript
import myImage from "../../assets/hero.png";
<img src={myImage} />;
```

### 3D Models (GLTF) - **CRITICAL**

Stake Engine **BLOCKS** `blob:` URLs. Standard GLTF loaders will fail.
You **MUST** use the Data URI patching technique.

1.  Put `.glb` files in `public/models/`.
2.  Use the `loadPatchedModel` function (ask me for the code if you don't have it, or refer to `Stake_Engine_3D_Guide.md`).
3.  **NEVER** put `.glb` files in `src/`.

---

## 9. Building for Production

When you are ready to upload to Stake Engine:

1.  **Check `vite.config.ts`**:
    Ensure `base: './'` is set so assets load with relative paths.

    ```typescript
    export default defineConfig({
      base: "./",
      plugins: [react()],
    });
    ```

2.  **Run Build**:

    ```bash
    npm run build
    ```

3.  **Verify Output**:
    Check the `dist` folder. It should contain `index.html` and an `assets` folder.

    - Open `dist/index.html` in a browser (you might need a local server like `npx serve dist`).
    - Ensure no 404 errors for assets.

4.  **Upload**: Zip the contents of `dist` and upload to Stake Engine.
