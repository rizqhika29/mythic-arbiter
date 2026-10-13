# Mythic Arbiter

> A **Decentralized Game Master** on GenLayer — players submit free-text actions each turn, an LLM judges them, and validator consensus decides the outcome, while every game rule (HP, XP, level, death, respawn) is computed deterministically inside the contract.

## Deploy Info

| | |
|---|---|
| **Network** | GenLayer Studio Network (studionet) |
| **Contract Address** | [`0x884aCED9C7551F1B5ba26d360D5D85bB3faF5E0b`](https://explorer-studio.genlayer.com/address/0x884aCED9C7551F1B5ba26d360D5D85bB3faF5E0b) |
| **Host Account** | `0xB02eA5D0d13B88D4cf5BE6Fbe88DB5835C4c1ad3` |
| **Guest Account** | `0x0481cE639B6059a8200f2484746aDC926A0B3E7C` |
| **Live Test** | **38/38 passed** — every write & view method, including one real AI-consensus turn resolution (verdicts `success`/`success`, XP 15/15) |
| **Unit Tests** | **101 passed** |

---

## How It Works

1. The host calls `create_campaign()` (title, setting, rules, max players, action window, grace period) — the caller becomes the host.
2. Players call `join_campaign()` — a character is created (HP 100, level 1).
3. The host calls `start_campaign()` — the campaign goes `active`, turn 1 opens.
4. Each player calls `submit_action()` (max one action per turn).
5. The host calls `close_action_window()` to lock the turn early (optional; the turn also locks automatically after `turn_deadline`).
6. `resolve_turn()` builds the Game Master prompt, calls the LLM, and validator consensus verifies the mechanical result.
7. The contract applies the mechanical result deterministically → the next turn opens.

---

## Consensus Design

The contract deliberately **limits the LLM to closed sets** so no consensus break can come from stylistic differences:

| Field | Closed Set |
|-------|------------|
| `verdict` | `critical_success`, `success`, `partial`, `failure` |
| `damage` | `0, 5, 10, 20, 40` |
| `heal` | `0, 5, 10, 20` |

Everything else is **derived deterministically by the contract**, never by the LLM:

- XP per verdict: `30 / 15 / 5 / 0`
- Level-up every 100 XP; `max_hp = 100 + 10 × (level − 1)`, max level 20
- HP moves by `damage`/`heal`; death at HP ≤ 0
- Death halves accumulated XP; `respawn_character()` revives at half `max_hp`
- Stats: `actions_taken`, `successes`, `failures`, `deaths`

### What validators compare

Validators compare the **material consequence** of each action, not the prose:

- `idx` must be identical (same set of action indices, each exactly once).
- `verdict` is compared by **success/failure class**: `critical_success`/`success` = success, `partial`/`failure` = failure. Two Game Masters often disagree on whether an action was a clean success or a partial one; that difference in generosity must not break consensus, but the success/failure boundary must.
- `damage` and `heal` are compared exactly (from the small closed sets).
- `reward` is compared by **presence** (granted or not), never by its text: item names can never be identical across two LLM calls, but *whether a reward exists* is exactly the part that reaches the character's inventory — so presence is consensus-bound while the label is not.
- `confidence` within a 25-point tolerance.

Narration (`scene`, `world_state`) with different wording is **not** a consensus mismatch — and is **never** fed back into the prompt (see below).

> **Best-effort by nature.** A Game Master is subjective; validators run different models. AI consensus **succeeds when models agree on the material consequence** of an action and **fails** on borderline calls (e.g. one model grades "success" while another grades "partial"). A failed consensus never wedges the campaign: `resolve_turn_timeout` always advances the turn deterministically after the grace period, so liveness is fully guaranteed by the escape hatch.

### Why narration is never fed back into the prompt

`scene`/`world_state` are written by the leader and cannot be validator-verified (two LLM calls never produce identical prose). If those values were fed back into the next turn's prompt, one rogue leader could poison the context of every subsequent turn without any validator ever attesting to it.

Therefore:

- Narration is **stored** for the UI (`get_scene`, `get_narrative_log`) but **never** becomes Game Master input.
- Its replacement is a deterministic, consensus-bound summary: `turn {N-1} judged {k} actions` or `voided`.
- All state that actually influences the next decision (HP, level, XP, `is_alive`) comes from mechanics already agreed by validators.

### Prompt-injection fence

Host and player text (`title`, `setting`, `rules`, `action_text`, character name/archetype) is sent to the model inside `<untrusted-data>` tags with an explicit instruction that the content is **data, not instructions**. "Ignore previous instructions" inside an action must be ignored. Bounds still hold even if the model complies: values outside the closed sets fail normalization and the turn reverts cleanly.

### Anti-liveness escape hatch

`resolve_turn_timeout(campaign_id)` is a **fully deterministic** path (no LLM). Anyone may call it after `turn_deadline + resolution_grace_sec` to void unresolved actions, increment `turns_voided`, and open the next turn.

A turn with **no actions at all** also advances deterministically (no LLM, no revert). Without this, a host closing the window before anyone acts — or an entire party wiped out and never respawned — would wedge the campaign forever.

### Action-window gate

The host may close the window early, but **not before `MIN_CLOSE_DELAY_SEC` (30 seconds)** have elapsed since the turn opened. Without this, a hostile host could run a `start → close` loop and players would never get a single action in.

---

## Contract Structure

### Data Models

```python
@dataclass
class Campaign:
    campaign_id: str
    host: Address
    title: str
    setting: str
    rules: str
    max_players: u32
    status: str                 # lobby | active | ended
    players: DynArray[Address]
    turn_number: u32
    turn_opened_at: u32
    turn_deadline: u32
    window_closed: bool
    action_window_sec: u32
    resolution_grace_sec: u32
    created_at: u32
    turns_resolved: u32
    turns_voided: u32
    total_actions: u32

@dataclass
class Character:
    campaign_id: str
    owner: Address
    name: str
    archetype: str
    hp: u32
    max_hp: u32
    level: u32
    xp: u32
    deaths: u32
    is_alive: bool
    is_active: bool
    actions_taken: u32
    successes: u32
    failures: u32
    inventory: DynArray[str]
    joined_at: u32

@dataclass
class GameAction:
    campaign_id: str
    turn: u32
    player: Address
    character_name: str
    action_text: str
    submitted_at: u32
    status: str                 # pending | resolved | void
    verdict: str
    outcome: str
    damage: u32
    heal: u32
    reward: str
    resolved_at: u32

@dataclass
class TurnNarrative:
    campaign_id: str
    turn: u32
    scene: str
    world_state: str
    actions_resolved: u32
    resolver: Address
    via_timeout: bool
    resolved_at: u32
```

### Input Bounds

| Constant | Value |
|----------|-------|
| `MAX_TITLE_CHARS` | 80 |
| `MAX_SETTING_CHARS` / `MAX_RULES_CHARS` | 3000 |
| `MAX_ACTION_CHARS` | 600 |
| `MAX_NAME_CHARS` / `MAX_ARCHETYPE_CHARS` | 32 |
| `MAX_PLAYERS_LIMIT` | 20 |
| `action_window_sec` | 60 … 30 days |
| `resolution_grace_sec` | 60 … 7 days |

---

## Methods

### Write Methods

| Method | Payable | Args | Description |
|--------|---------|------|-------------|
| `create_campaign` | No | `title, setting, rules, max_players, action_window_sec, resolution_grace_sec` | Open a new campaign; caller becomes host |
| `join_campaign` | No | `campaign_id, character_name, archetype` | Join a campaign (still lobby) and create a character |
| `leave_campaign` | No | `campaign_id` | Leave while still in lobby (host cannot leave) |
| `start_campaign` | No | `campaign_id` | Host only. Lobby → active, open turn 1 |
| `update_campaign_rules` | No | `campaign_id, rules` | Host only, lobby only. Rules frozen after start |
| `end_campaign` | No | `campaign_id` | Host only. End campaign, void pending actions |
| `submit_action` | No | `campaign_id, action_text` | Submit a free-text action for the current turn (max 1/turn) |
| `close_action_window` | No | `campaign_id` | Host only. Lock the turn so it can be judged |
| `resolve_turn` | No | `campaign_id` | Call the Game Master (LLM) + validator consensus, apply the result |
| `resolve_turn_timeout` | No | `campaign_id` | Deterministic LLM-free path after the grace period |
| `respawn_character` | No | `campaign_id` | Revive a dead character (half max HP, half XP) |

### View Methods

| Method | Args | Description |
|--------|------|-------------|
| `get_campaign` | `campaign_id` | Campaign details |
| `list_campaigns` | — | Summary of all campaigns |
| `get_character` | `campaign_id, owner` | Character details for a specific address |
| `get_characters` | `campaign_id` | All characters in a campaign |
| `get_pending_actions` | `campaign_id` | Current-turn actions not yet judged (with `idx`) |
| `get_turn_actions` | `campaign_id, turn` | All actions on a given turn with their results |
| `get_scene` | `campaign_id` | Current scene & world state |
| `get_narrative_log` | `campaign_id` | Narrative log (latest 25 chapters) |
| `get_contract_address` | — | This contract's address |
| `get_stats` | — | `admin`, `total_campaigns`, `total_actions`, `total_narrative_entries` |

---

## Campaign Lifecycle

```
[lobby] --start--> [active] --end--> [ended]
                     |
                     +-- per turn: open → submit → close → resolve
                                             \→ (grace elapsed) resolve_turn_timeout
```

- **lobby** — free join / leave / edit rules
- **active** — actions only accepted before `turn_deadline` and before `window_closed`
- **ended** — terminal; all mutation calls are rejected

---

## Security & Anti-Rejection

1. **No escrow (non-payable)** — no fund/liveness gap involving money.
2. **Closed value sets** — the LLM only picks verdict/damage/heal from fixed enums; everything else is computed by the contract.
3. **Material-consequence comparison** — validators compare `idx` (identical), `verdict` by success/failure class, `damage`/`heal` exactly, and `reward` by presence. Narration is never compared (two LLMs can never agree on it) and is never fed back into the prompt, so there is no unverified influence channel.
4. **`reward` is consensus-bound** — the *presence* of a reward (granted or not) must match between leader and validator, because that is what determines whether a character's inventory grows. The item label itself stays leader-committed (length-bounded).
5. **Deterministic escape hatch** — `resolve_turn_timeout` defeats the liveness gap; campaigns cannot wedge.
6. **Empty turns still advance** — a turn with no actions advances deterministically with no LLM, so no permanent deadlock.
7. **Dual action window** — closed by the host *or* automatically via `turn_deadline`; the host cannot close before `MIN_CLOSE_DELAY_SEC`.
8. **Caller restrictions** — start/end/close/rules are host-only; actions are member-only; respawn is only for your own dead character.
9. **Bounded inputs** — all text is length-capped; `max_players` and time windows are validated before storage.
10. **Bounded prompt data** — `scene` (800), `world_state` (500), `reward` (120), inventory (10 items).
11. **Prompt-injection fence** — all host/player text is sent inside `<untrusted-data>` with an instruction that it is data, not instructions.
12. **Guarded status transitions** — actions cannot enter after the window closes; rules freeze after start; campaigns end cleanly.
13. **Bounded per-turn work** — the campaign keeps its own pending-action index list, so submit/resolve cost O(players), not O(all actions ever).

---

## Error Handling

All contract errors use `gl.vm.UserError` with an `[EXPECTED]` prefix:

**General / input validation**

- `[EXPECTED] {label} must not be empty`
- `[EXPECTED] {label} too long (max {limit} chars)`
- `[EXPECTED] campaign not found`
- `[EXPECTED] character not found in this campaign`
- `[EXPECTED] max_players must be between 1 and 20`
- `[EXPECTED] action_window_sec must be between 60 and 2592000`
- `[EXPECTED] resolution_grace_sec must be between 60 and 604800`

**Lobby**

- `[EXPECTED] campaign is not accepting new players`
- `[EXPECTED] campaign is full`
- `[EXPECTED] already joined this campaign`
- `[EXPECTED] can only leave while the campaign is in the lobby`
- `[EXPECTED] the host cannot leave their own campaign`
- `[EXPECTED] you have not joined this campaign`
- `[EXPECTED] only the host can start the campaign`
- `[EXPECTED] campaign is not in the lobby`
- `[EXPECTED] at least one player must join before starting`
- `[EXPECTED] only the host can edit the rules`
- `[EXPECTED] rules are frozen once the campaign has started`
- `[EXPECTED] only the host can end the campaign`
- `[EXPECTED] campaign is already ended`

**Action window & turn resolution**

- `[EXPECTED] campaign is not active`
- `[EXPECTED] only joined players can act`
- `[EXPECTED] your character is defeated; respawn first`
- `[EXPECTED] you already submitted an action this turn`
- `[EXPECTED] action window is already closed`
- `[EXPECTED] action window is closed (deadline {turn_deadline})`
- `[EXPECTED] only the host can close the action window`
- `[EXPECTED] action window still open until {deadline}`
- `[EXPECTED] grace period not elapsed; timeout available after {deadline}`
- `[EXPECTED] turn resolution indeterminate ({reason}) - retry later or call resolve_turn_timeout after the grace period`
- `[EXPECTED] only joined players can respawn`
- `[EXPECTED] your character is already alive`

---

## Tests

### Direct tests

```bash
$env:PRIVATE_KEY_1="0x...01"; $env:PRIVATE_KEY_2="0x...02"; $env:TESTNET_PRIVATE_KEY="0x...03"
python -m pytest tests/direct/ -q
```

Covers the campaign lifecycle, input validation, action-window gates, consensus resolution (mocked LLM), validator mismatch cases (including `reward`), consensus tolerance (different reward wording but same presence, same-class verdicts), the timeout escape hatch, empty turns, prompt-injection fencing, and every view. **101 tests, all passing.**

### Integration tests (against a deployment)

```bash
$env:MYTHIC_ARBITER_ADDRESS="0x..."
python -m pytest tests/integration/
```

### Live tests on GenLayer Studio

```bash
python tools/live_test.py 0x<contract_address>
```

Runs every write method over the real RPC, including one turn resolution with AI consensus.

---

## Deploy

```bash
python tools/deploy.py
```
