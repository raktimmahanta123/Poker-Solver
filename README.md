# Build a Real‑Time Poker Solver UI (GTO Wizard‑like) — Full Spec for GitHub Copilot

You are GitHub Copilot. Generate a production‑ready full‑stack TypeScript app that runs a **real‑time, GTO‑Wizard‑style poker solver UI** with an extensible rules engine (stubbed) that adapts preflop and postflop decisions based on **game type, tournament stage, positions, stacks, player types, and spot**. The app should be fast, type‑safe, and easy to extend with a real solver later.

---

## Tech Stack

* **Frontend:** Next.js (App Router) + React + TypeScript
* **Styling:** Tailwind CSS + shadcn/ui + Radix primitives
* **Icons & Motion:** lucide-react + framer-motion
* **State Management:** Zustand (store slices) + URL query sync (next/navigation)
* **Forms/Validation:** Zod + react-hook-form
* **Server:** Next.js Route Handlers (/app/api/\*) with TypeScript
* **Testing:** Vitest + React Testing Library + Playwright (basic flows)
* **Lint/Format:** ESLint (strict) + Prettier
* **Build/Deploy:** Vercel‑friendly; `.nvmrc`, `tsconfig`, `vercel.json`

Create a **monorepo‑style folder layout** inside a single Next.js app for clarity: `app/`, `components/`, `lib/`, `data/`, `store/`, `styles/`, `tests/`.

---

## Core Concept

We are **not** building a heavy solver in this step. Implement a **pluggable `SolveEngine`** that returns decisions and ranges using rules + JSON data. Later we can swap it with a proper engine. Keep performance snappy and UI state pure.

---

## Domain Model & Types (TypeScript)

Define strong types in `lib/types.ts`:

```ts
export type GameMode = "MTT" | "Cash";

export type TournamentStage =
  | "Early"
  | "ICM_50_to_20" // 50% to 20%
  | "Money_Bubble"
  | "Post_Bubble_Early"
  | "Post_Bubble_Mid"
  | "Post_Bubble_Late"
  | "Final_Two_Tables"
  | "FT_Bubble"
  | "Final_Table";

export type Position = "UTG" | "UTG1" | "LJ" | "HJ" | "CO" | "BTN" | "SB" | "BB"; // Accept alias "BT" for BTN in parsers/display

export type PlayerType = "TAG" | "LAG" | "LP" | "TP" | "Unknown"; // Tight/Loose Passive

export type Spot =
  | "RFI"
  | "Facing_Open"
  | "Facing_3Bet"
  | "Facing_4Bet"
  | "Facing_Limp"
  | "Facing_AllIn"
  | "Blind_vs_Blind";

export type PotKind = "SRP" | "ThreeBet" | "FourBet" | "AllLimp";

export type Action =
  | "Check"
  | "Fold"
  | "Call"
  | "Raise"
  | "Bet_Min"
  | "Bet_25"
  | "Bet_33"
  | "Bet_50"
  | "Bet_66"
  | "Bet_Pot"
  | "Check_Raise";

export type ComboAction =
  | "Fold_Blue"
  | "Call_Green"
  | "Raise_Orange"
  | "ThreeBet_Value_Red"
  | "ThreeBet_Bluff_Purple"
  | "FourBet_Pink"
  | "FourBet_Bluff_Yellow"
  | "AllIn_LightBrown";

export type StackBB = number; // always in big blinds

export interface PreflopContext {
  gameMode: GameMode;
  stage?: TournamentStage; // required when gameMode=MTT
  heroPos: Position;
  spot: Spot;
  vsPos?: Position; // depends on spot
  heroStackBB: StackBB; // effective stack vs villain for simplicity
  villainStackBB?: StackBB;
  villainType: PlayerType; // adapts ranges exploitatively
}

export interface PostflopContext {
  potKind: PotKind;
  street: "Flop" | "Turn" | "River";
  flop?: { c1: string; c2: string; c3: string }; // e.g., "As","Ts","Kd"
  toAct: "Hero" | "Villain";
  potBB: number; // pot in big blinds
  spr: number;   // stack to pot ratio
  history: Array<{ actor: "Hero" | "Villain"; action: Action; sizeBB?: number }>; // legal sequence only
}

export interface RangeCell {
  hand: string; // e.g., "AKs", "A5o"
  freq: number; // 0..1
  label: ComboAction; // color category
}

export interface SolveResult {
  range13x13: RangeCell[][]; // 13x13 grid, row=rank, col=rank
  legend: Record<ComboAction, string>;
  note?: string;
}

export interface Decision {
  best: Action; // highest EV action (or mix -> choose top EV)
  sizeBB?: number; // if a bet/raise
  alt?: Array<{ action: Action; sizeBB?: number; ev: number }>; // optional
}
```

---

## Data & Heuristics

In `data/`, provide JSON stubs the engine can use:

* `preflop/baseRanges.json`: baseline GTO‑like ranges per (gameMode, stage?, heroPos, spot, stack bucket) with frequencies (0..1). Use realistic placeholders.
* `preflop/adjustments.json`: rules to tilt frequencies based on `villainType` & `stage` (e.g., vs **LAG** → narrow calling, increase 3B value & 4B jam; vs **LP** → open wider, iso more, c‑bet more; **ICM** → tighten, reduce bluff 4B, prefer flatting, etc.).
* `postflop/sizeMenu.json`: allowed bet sizes per street: `Min`, `25%`, `33%`, `50%`, `66%`, `Pot` (all in BB terms via `potBB`).

Create a helper `lib/weights.ts` to map **stack buckets** (e.g., `<=20bb`, `21–40bb`, `41–80bb`, `>80bb`) and **ICM stages** to risk adjustments.

---

## Solver Engine (Pluggable)

Create `lib/engine/solveEngine.ts` with an exported `SolveEngine` interface and a default rules implementation:

* `solvePreflop(ctx: PreflopContext): Promise<SolveResult>`

  * Merge `baseRanges` + `adjustments` by `villainType`, `stage`, and `stack bucket`.
  * Enforce **spot legality** (see Constraints below). If illegal, return helpful error.
  * Return a **13×13 matrix** with combo labels and frequencies.
* `solvePostflop(preCtx: PreflopContext, postCtx: PostflopContext): Promise<Decision>`

  * Only return **legal** actions based on `history` and `toAct` (e.g., no check‑raise unless there was a prior check then bet).
  * Compute **best action** using heuristics: board texture, position, potKind, SPR, villainType, stage; allow **overbets later** (future extension) but for now include sizes specified.
  * Always express **sizes in BB** (convert from pot%).

Keep EVs mocked but consistent: higher EV for actions aligned with TAG vs villainType/stage and texture.

---

## Constraints & Dropdown Logic

Implement **dependent dropdowns** with hard validations in `lib/constraints.ts` and UI guards:

* **RFI** → `vsPos` hidden (N/A).
* **Facing\_Open / Facing\_3Bet / Facing\_4Bet / Facing\_AllIn** → `vsPos` must be **earlier** than `heroPos` in table order (e.g., **LJ** cannot face open from **BTN**; **UTG** cannot face open from **SB/BB**).
* **Facing\_Limp** → `vsPos` is limper’s seat; must be **earlier** than `heroPos`.
* **Blind\_vs\_Blind** → restrict `heroPos` ∈ {SB, BB}, hide `vsPos` and force it to the other blind.
* **Stack sanity**: `heroStackBB, villainStackBB` ∈ \[1, 400]. Effective stack = `min(heroStackBB, villainStackBB)` when both set.
* Disable impossible actions at every point (e.g., no `Check_Raise` without prior `Check` then `Bet`).

Add a table order array to compare seat order: `["UTG","UTG1","LJ","HJ","CO","BTN","SB","BB"]`.

---

## UI Requirements

### 1) Preflop Panel

* Controls: `GameMode`, `TournamentStage` (enabled if `MTT`), `Hero Position`, `Spot`, `Vs Position` (contextual), `Hero Stack (bb)`, `Villain Stack (bb)`, `Villain Type`.
* All controls are **shadcn/ui** selects/number inputs with validation, animated with **framer‑motion**.
* **13×13 Range Matrix** component (`<RangeMatrix />`):

  * Ranks from A → 2 axes; combos cells labelled per `ComboAction` and painted with required colors:

    * Call = **Green**
    * Fold = **Blue**
    * Raise = **Orange**
    * 3Bet Value = **Red**
    * 3Bet Bluff = **Purple**
    * 4Bet = **Pink**
    * 4Bet Bluff = **Yellow**
    * All‑in = **Light Brown**
  * Show **frequency** on hover; optional tiny pie to reflect mix.
  * Legend below with the same color chips.
* Below matrix: **PotKind selector**: `SRP`, `3Bet`, `4Bet`, `All‑Limp`.

### 2) Postflop Panel

* **Flop input**: dual mode

  1. Three dropdowns (Rank, Suit) for each card.
  2. Free text like `AsTsKd` or spaced `As Ts Kd` with robust parser.
* **Actor toggle**: `Villain to act` or `Hero to act`.
* **Villain action menu** (only when Villain to act): show **only viable** options among `Check`, `Bet_Min`, `Bet_25`, `Bet_33`, `Bet_50`, `Bet_66`, `Bet_Pot`; `Fold` appears **only when facing a bet**; `Check_Raise` appears **only** after a prior `Check` then `Bet`. Add a settings toggle to enable **Overbet** sizes (`Bet_125`, `Bet_150`, `Bet_200`) for future use.
* After user selects villain action, **auto‑display Hero’s highest‑EV action** with size (BB). If the line continues (raises etc.), prompt next villain action accordingly; keep enforcing sequence legality.
* Show **pot (BB)**, **SPR**, and a compact **action log** with badges.

### 3) UX niceties

* **URL state sync** for easy share links.
* **Keyboard shortcuts** for moving streets/actions.
* **Dark mode** default.
* **A11y**: aria labels, focus rings, high contrast colors.
* **Toasts** for illegal selections with helpful text.

---

## Colors & Styling

Add Tailwind CSS variables in `styles/globals.css` for the 8 categories and expose via a `lib/colors.ts` map:

```ts
export const ComboColors: Record<ComboAction, string> = {
  Call_Green: "bg-green-500",
  Fold_Blue: "bg-blue-500",
  Raise_Orange: "bg-orange-500",
  ThreeBet_Value_Red: "bg-red-600",
  ThreeBet_Bluff_Purple: "bg-purple-500",
  FourBet_Pink: "bg-pink-400",
  FourBet_Bluff_Yellow: "bg-yellow-300",
  AllIn_LightBrown: "bg-amber-700", // light brown approximation
};
```

Use soft shadows, rounded‑2xl cards, grid layouts, and motion fades.

---

## File/Folder Outline

```
app/
  layout.tsx
  page.tsx                      // Preflop + Postflop panels
  api/
    solve/
      preflop/route.ts          // POST { PreflopContext } -> SolveResult
      postflop/route.ts         // POST { PreflopContext, PostflopContext } -> Decision
components/
  controls/
    GameModeSelect.tsx
    StageSelect.tsx
    PositionSelect.tsx
    SpotSelect.tsx
    VsPositionSelect.tsx
    StackInput.tsx
    PlayerTypeSelect.tsx
    PotKindSelect.tsx
    FlopInput.tsx
    ActorToggle.tsx
    VillainActionMenu.tsx
  matrix/
    RangeMatrix.tsx
    Legend.tsx
  layout/
    Card.tsx (wrap shadcn Card)
    Toolbar.tsx
  readouts/
    PotSPRBar.tsx
    ActionLog.tsx
lib/
  types.ts
  constraints.ts
  engine/
    solveEngine.ts
    heuristics.ts
    textures.ts // board texture helpers
  colors.ts
  weights.ts
  util.ts (parsers, card utils, BB converters)
store/
  uiStore.ts (Zustand slices: preflop, postflop, flow)
  historyStore.ts
  urlSync.ts
data/
  preflop/
    baseRanges.json
    adjustments.json
  postflop/
    sizeMenu.json
styles/
  globals.css
  shadcn.css
tests/
  unit/
    constraints.test.ts
    utilParser.test.ts
    enginePreflop.test.ts
  e2e/
    basicFlow.spec.ts (Playwright)
```

---

## API Contracts

**POST /api/solve/preflop**

* Input: `PreflopContext`
* Output: `SolveResult`

**POST /api/solve/postflop**

* Input: `{ pre: PreflopContext, post: PostflopContext }`
* Output: `Decision`

Return 400 with a descriptive `error` for illegal contexts.

---

## Constraints Implementation Details

* A reusable `isEarlier(posA, posB)` per seat order.
* Validation guards in UI + server: RFI has no `vsPos`; Facing\_\* require `vsPos` earlier than `heroPos`; BvB only SB vs BB.
* Postflop legality:

  * `Check_Raise` only after `[Check -> Bet_*]` in `history` and when toAct matches the checker.
  * Prevent double actions; keep turn order strict.
  * Enforce min/allowed raise sizes using pot math; expose helper `legalActions(state)`.

---

## Heuristic Rules (initial)

* **TAG baseline** with adaptive knobs:

  * vs **LAG**: tighten flats, increase 3B value, reduce 3B bluffs OOP, increase 4B for value; postflop more check‑raise for value, fewer floats OOP.
  * vs **LP**: widen RFI/iso, increase c‑bets small, barrel scare cards; add more thin value; fewer bluff 4Bs.
  * vs **TP**: exploit overfolds; add polarized 3B bluffs IP; increase delayed c‑bets.
  * **ICM stages**: reduce variance; fewer bluff 4Bs; more flats; tighten opening esp. near bubbles; avoid thin bluffs multiway.
* **Stack buckets**: at `<=20bb`, prioritize jam/fold trees; at deep stacks, include 4B bluff mixes; prefer BB calls with suited/connected holdings.

Document these in `data/preflop/adjustments.json` and reference within the engine.

---

## Range Matrix Rendering

* 13×13 grid with ranks A→2 top and left.
* Mixed strategies: choose **dominant label** by highest frequency bucket; show tooltip with full mix.
* Click a cell to show combo breakdown panel.
* Legend with the 8 color chips and labels.

---

## Postflop Flow

1. User selects **PotKind** → compute baseline potBB for preflop line.
2. Input **Flop** via dropdowns or text (parse both), choose **toAct**.
3. If **Villain to act** → display only legal actions and bet sizes.
4. On villain action confirm → call `/api/solve/postflop` → show **Hero best action** (EV highest) with size in **BB**.
5. If action continues (raises) → advance the state machine and continue prompting villain.
6. If **Hero to act** initially → show Hero best action immediately.

---

## README.md (generate)

* Project intro & screenshots.
* How to run locally (`pnpm i && pnpm dev`).
* Explanation of **dependent dropdown rules** and **legality checks**.
* Data stubs and how to plug a real solver later.
* Color legend and UX tips.

---

## Tests (generate)

* Unit tests for:

  * `constraints.ts` (RFI vs vsPos rules, BvB, seat order)
  * `util.ts` card parser (`AsTsKd`, `Kd Qc 7h` variants)
  * `enginePreflop` merges base + adjustments deterministically
* E2E:

  * Preflop → Range matrix renders & legend visible.
  * Postflop villain bet → hero best action appears with BB size.

---

## Delivery Checklist

* All files created with meaningful comments.
* Strong TypeScript types throughout.
* shadcn/ui components installed and configured.
* Dark mode, motion, and a crisp UI.
* No placeholder TODOs in critical paths.
* App boots with seed data and is usable immediately.

---

## Stretch Goals (optional if time permits)

* Import/export ranges as JSON.
* Frequency sliders to nudge exploit adjustments.
* Turn/River support with board runouts and blockers awareness.
* Shareable permalink of current state.

---

**Build the entire app now per the spec above.** Create all files, components, data stubs, and tests so it runs out of the box.
