# Tasks: Desktop Chess Client

**Feature**: `001-desktop-chess-client`  
**Input**: Design documents from `/specs/001-desktop-chess-client/`  
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/server-api.md ✅, quickstart.md ✅

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2…)

---

## Phase 1: Setup

**Purpose**: Project initialization, directory scaffolding, and tooling configuration

- [ ] T001 Create project directory structure: `src/ui/screens/`, `src/ui/components/`, `src/state/`, `src/adapter/`, `src/models/`, `src/assets/sprites/pieces/`, `src/assets/sprites/ui/`, `src/assets/sprites/board/`, `src/assets/fonts/`, `tests/unit/`, `tests/contract/`, `tests/integration/`
- [ ] T002 Create `requirements.txt` with pinned dependencies: `pygame==2.1.2`, `requests==2.28.0`, `python-dotenv==0.20.0`, `pytest==7.2.0`, `mypy==0.990`, `black==22.12.0`
- [ ] T003 [P] Create `.env.example` with documented environment variables: `SERVER_URL`, `POLLING_INTERVAL`, `DEBUG`, `WINDOW_WIDTH`, `WINDOW_HEIGHT`
- [ ] T004 [P] Create `pyproject.toml` configuring `[tool.mypy]` (strict mode, source root `src/`) and `[tool.black]` (line-length 100)
- [ ] T005 [P] Create `pytest.ini` with `testpaths = tests`, `markers = unit, contract, integration`
- [ ] T006 [P] Create `src/__init__.py`, `src/ui/__init__.py`, `src/ui/screens/__init__.py`, `src/ui/components/__init__.py`, `src/state/__init__.py`, `src/adapter/__init__.py`, `src/models/__init__.py` (empty module markers)
- [ ] T007 [P] Create `tests/__init__.py`, `tests/unit/__init__.py`, `tests/contract/__init__.py`, `tests/integration/__init__.py`

**Checkpoint**: Directory structure and tooling in place; `pytest` discovers `tests/` without errors

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure required before any user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T008 Create all domain models in `src/models/game.py`: `GameStatus` enum (`open`, `active`, `finished`), `GameResult` enum (`white_wins`, `black_wins`, `draw`, `in_progress`), `GameOutcome` dataclass, `GameSnapshot` dataclass (game_id, status, white_player, black_player, fen, move_history, active_color, outcome, created_at)
- [ ] T009 [P] Create session model in `src/models/session.py`: `Session` dataclass (session_id, username, created_at)
- [ ] T010 [P] Create move model in `src/models/move_command.py`: `Piece` enum (`q`, `r`, `b`, `n`), `Move` dataclass (from_square, to_square, promotion), `to_uci()` method
- [ ] T011 [P] Create UI event types in `src/models/ui_event.py`: `ClickEvent`, `KeyEvent`, `UIEvent` dataclasses; `SquareCoord` dataclass; `BoardOrientation` enum (`WHITE`, `BLACK`)
- [ ] T012 [P] Create HTTP response types in `src/models/api_types.py`: `SessionResponse`, `GameSummaryResponse`, `GameDetailResponse`, `MoveResponse`, `ErrorResponse` dataclasses matching server JSON contracts from `contracts/server-api.md`
- [ ] T013 Create `src/adapter/http_client.py`: base HTTP client class wrapping `requests.Session`; reads `SERVER_URL` from env; attaches `X-Session-ID` header when session is set; handles connection errors, timeouts (10s default), and 5xx errors by raising typed `ServerError` exceptions; logs requests when `DEBUG=true`
- [ ] T014 [P] Create `src/adapter/error_handling.py`: `ServerError` exception class; `parse_error_response(response)` → `ServerError`; `ERROR_MESSAGES` dict mapping server error codes (e.g., `game_not_open`, `wrong_turn`) to user-facing toast strings from `contracts/server-api.md`
- [ ] T015 Create application state machine in `src/state/state_machine.py`: `AppState` enum (all 9 states from `data-model.md`); `ApplicationState` dataclass; `StateMachine` class with `transition(event)` method; valid transitions table matching `data-model.md` state diagram; raises `InvalidTransitionError` for illegal transitions
- [ ] T016 Create `src/ui/theme.py`: `PALETTE` dict with named colours (16–32 muted pixel-art colours); `BOARD_LIGHT`, `BOARD_DARK` colours; `FONT_PATH`, `FONT_SIZE_SM/MD/LG` constants; `WINDOW_WIDTH`, `WINDOW_HEIGHT` constants (default 1024×768); helper `load_font(size)` using `pygame.font`
- [ ] T017 Create `src/main.py`: initialise `pygame`; load `.env` via `python-dotenv`; create `pygame.display` window using theme constants; instantiate `StateMachine`; start main event loop (process `pygame.event` queue → dispatch to active screen); call `pygame.quit()` on exit

**Checkpoint**: App launches to a blank window with no errors; state machine transitions can be tested in isolation; HTTP client can be instantiated

---

## Phase 3: User Story 1 – Enter Username and Access Lobby (Priority: P1) 🎯 MVP

**Goal**: User can enter a username, the client creates a server session, and the lobby screen appears with the game list.

**Independent Test**: Launch app, enter username "Alice", click Play — confirm lobby screen shows with an empty-state message (no games). Delivers a runnable app shell before any game logic is needed.

### Implementation

- [ ] T018 [US1] Create `src/adapter/session_api.py`: `create_session(http_client, display_name) -> Session`; calls `POST /sessions`; returns `Session`; raises `ServerError` on HTTP error; raises `ValidationError` if `display_name` is empty or > 32 chars
- [ ] T019 [P] [US1] Create `src/adapter/games_api.py` stub: `list_games(http_client) -> list[GameSummaryResponse]`; calls `GET /games`; returns list (may be empty)
- [ ] T020 [US1] Create `src/ui/components/button.py`: `PixelArtButton` class (rect, label, colour from theme); `draw(surface)` method; `is_hovered(pos)`, `is_clicked(pos, event)` methods; hover state changes colour to palette highlight
- [ ] T021 [P] [US1] Create `src/ui/components/text_input.py`: `PixelArtTextInput` class (rect, placeholder, max_length=32); `draw(surface)` method; `handle_event(event)` updates internal text buffer; `value` property; validates printable chars only
- [ ] T022 [P] [US1] Create `src/ui/components/toast.py`: `Toast` dataclass (message, action_label, action_type, created_at); `ToastManager` class; `show(message, action=None)` adds toast; `update(dt)` removes toasts after 3–5 seconds; `draw(surface)` renders toast stack in bottom-right corner
- [ ] T023 [US1] Create `src/ui/screens/username_entry.py`: `UsernameEntryScreen` class; draws atmospheric pixel-art background (solid colour fill or gradient as placeholder until sprites created); renders `PixelArtTextInput` and `PixelArtButton("Play")`; on confirm, validates username (non-empty, ≤32 printable chars), dispatches `submit_username` event to state machine; shows inline error hint if invalid
- [ ] T024 [US1] Implement `src/adapter/games_api.py` `list_games()`: full implementation; parse `GameSummaryResponse` list; handle empty array; raise `ServerError` on 401/5xx
- [ ] T025 [US1] Create `src/ui/screens/lobby.py`: `LobbyScreen` class; renders game list from `ApplicationState.lobby_games`; shows "No open games" empty-state when list is empty; distinguishes open (joinable) from active/finished games; renders "New Game" button; calls `ToastManager.draw()` for error toasts
- [ ] T026 [US1] Wire `UsernameEntryScreen` → session creation → `LobbyScreen` in `src/main.py`: on `submit_username` event, call `session_api.create_session()`, store `session_id` in `ApplicationState`, transition state to `FETCHING_LOBBY`, fetch lobby, transition to `IN_LOBBY`, render `LobbyScreen`; on `ServerError`, show toast with retry action
- [ ] T027 [P] [US1] Create `src/ui/components/spinner.py`: `Spinner` class; animated 4-frame pixel-art spinner (rectangles rotating); `update(dt)` advances frame; `draw(surface, x, y)` renders current frame; used during any async fetch

**Checkpoint**: App launches → username screen → enter name → lobby appears with empty game list or server games displayed

---

## Phase 4: User Story 2 – View and Join Open Games (Priority: P2)

**Goal**: User sees open games in lobby and can join one, transitioning to the game board.

**Independent Test**: With one open game already on server, second user launches app, enters username, sees game in lobby, clicks Join — board screen appears.

### Implementation

- [ ] T028 [US2] Implement lobby polling in `src/state/poller.py`: `LobbyPoller` class; background thread; calls `games_api.list_games()` every `POLLING_INTERVAL` seconds (default 1.5); updates `ApplicationState.lobby_games` on success; calls `on_error(msg)` callback on `ServerError`; `start()` / `stop()` thread lifecycle methods
- [ ] T029 [US2] Update `src/ui/screens/lobby.py`: add per-game row rendering with creator name and status badge; add "Join" button for open games the current user did not create; "Join" button dispatches `join_game(game_id)` event; show `Spinner` while join request is in-flight
- [ ] T030 [US2] Implement `src/adapter/games_api.py` `join_game(http_client, game_id) -> GameDetailResponse`; calls `POST /games/{id}/join`; returns game detail; maps `409` → `"Game already started"` toast; maps `403` → `"Cannot join your own game"` toast; maps `404` → `"Game not found"` toast
- [ ] T031 [US2] Implement `src/adapter/games_api.py` `get_game(http_client, game_id) -> GameDetailResponse`; calls `GET /games/{id}`; returns full game snapshot
- [ ] T032 [US2] Create `src/state/game_state.py`: `build_game_snapshot(response: GameDetailResponse) -> GameSnapshot`; converts API response to domain `GameSnapshot`; determines `player_color` based on `session_id` vs `white_player`/`black_player`
- [ ] T033 [US2] Wire join flow in `src/main.py`: on `join_game` event, transition to `FETCHING_BOARD`, call `games_api.join_game()`, store `GameSnapshot` in `ApplicationState`, transition to `PLAYING_GAME`, render `BoardScreen`; on `ServerError`, show toast and return to `IN_LOBBY`
- [ ] T034 [US2] Update `LobbyScreen` to start `LobbyPoller` on entry and stop on exit; refresh game list from `ApplicationState` on each `POLLING_INTERVAL` tick

**Checkpoint**: Two clients can see each other's open games in the lobby; second client can join; both transition to board screen

---

## Phase 5: User Story 3 – Create a New Game (Priority: P2)

**Goal**: User can create a new game and see a waiting screen; both clients auto-transition when the second player joins.

**Independent Test**: User clicks "New Game", waiting screen appears with spinner and "Waiting for opponent…" text. A second client joins; both transition to board without manual action.

### Implementation

- [ ] T035 [US3] Implement `src/adapter/games_api.py` `create_game(http_client) -> GameSummaryResponse`; calls `POST /games`; returns game summary; raises `ServerError` on failure
- [ ] T036 [US3] Implement `src/adapter/games_api.py` `cancel_game(http_client, game_id) -> None`; calls `DELETE /games/{id}`; raises `ServerError` on failure (403 if not owner, 404 if not found)
- [ ] T037 [US3] Create `src/ui/screens/waiting.py`: `WaitingScreen` class; renders `Spinner`; shows "Waiting for opponent…" text; renders "Cancel" button; dispatches `cancel_game` event on cancel click; polls game status automatically
- [ ] T038 [US3] Implement waiting-screen poller in `src/state/poller.py`: `GamePoller` class (reusable for both waiting and active game); background thread; calls `games_api.get_game()` every `POLLING_INTERVAL` seconds; invokes `on_opponent_joined(snapshot)` callback when game status changes from `open` → `active`; `start()` / `stop()` lifecycle
- [ ] T039 [US3] Wire create-game flow in `src/main.py`: on "New Game" button click, call `games_api.create_game()`, store game in `ApplicationState`, transition to `WAITING_FOR_OPP`, render `WaitingScreen`, start `GamePoller`; on `opponent_joined`, transition to `PLAYING_GAME`, render `BoardScreen`; on `cancel_game`, call `cancel_game()`, transition to `IN_LOBBY`, render `LobbyScreen`

**Checkpoint**: Creator sees waiting screen; second player joins from lobby; both clients auto-navigate to board

---

## Phase 6: User Story 4 – Play a Chess Game (Priority: P1)

**Goal**: Two players can play a full chess game on the board screen with correct board orientation, move submission, and real-time updates.

**Independent Test**: Two clients in an active game alternate moves until checkmate. Board updates on both sides within one polling cycle. Game-over overlay appears with correct result.

### Implementation

- [ ] T040 [US4] Implement FEN board parser in `src/models/ui_event.py`: `fen_to_board(fen: str) -> list[list[Optional[str]]]`; parses FEN rank/file notation into 8×8 array; maps piece codes (`K`, `q`, `p`, etc.) to sprite keys (e.g., `"wK"`, `"bq"`)
- [ ] T041 [US4] Implement `screen_to_square(x, y, orientation, board_rect) -> Optional[SquareCoord]`; converts pixel click coordinates to board square (`row`, `col`); flips coordinates when `orientation == BLACK`; returns `None` if click is outside board area
- [ ] T042 [US4] Create `src/ui/components/board_widget.py`: `BoardWidget` class; renders 8×8 grid using `BOARD_LIGHT`/`BOARD_DARK` theme colours (placeholder until sprites created); overlays piece sprites from `fen_to_board()`; highlights selected square; highlights valid move destinations (translucent overlay); handles `MOUSEBUTTONDOWN`/`MOUSEBUTTONUP` events for click-to-select-then-click-to-place; emits `move_selected(Move)` callback
- [ ] T043 [P] [US4] Create `src/ui/components/piece_sprite.py`: `PieceSprite` class; loads 12 PNG files from `src/assets/sprites/pieces/` (`wp.png`, `wn.png`, `wb.png`, `wr.png`, `wq.png`, `wk.png`, `bp.png`, `bn.png`, `bb.png`, `br.png`, `bq.png`, `bk.png`); `draw(surface, piece_code, rect)` method; falls back to coloured rectangle placeholder if PNG not found (for development before assets created)
- [ ] T044 [P] [US4] Create `src/ui/components/promotion_dialog.py`: `PromotionDialog` class; renders modal overlay showing 4 piece buttons (queen, rook, bishop, knight); blocks board input until selection; fires `promotion_chosen(Piece)` callback; pixel-art styled consistent with theme
- [ ] T045 [US4] Create `src/ui/screens/board.py`: `BoardScreen` class; composes `BoardWidget`, `PieceSprite`, `PromotionDialog`, resign button, draw-claim button, `ToastManager`; reads `ApplicationState.current_game` and `ApplicationState.player_color`; passes correct `BoardOrientation` to `BoardWidget`; disables board interaction when it is not the player's turn; shows "Your turn" / "Opponent's turn" indicator
- [ ] T046 [US4] Implement `src/adapter/moves_api.py`: `submit_move(http_client, game_id, notation) -> MoveResponse`; calls `POST /games/{id}/moves`; maps `400` → `"Invalid notation"` toast; maps `422` → `"Illegal move"` toast; maps `403` → `"Not your turn"` toast; maps `409` → `"Game already over"` toast
- [ ] T047 [P] [US4] Implement `src/adapter/moves_api.py` `resign(http_client, game_id) -> dict`; calls `POST /games/{id}/resign`; returns response; raises `ServerError` on failure
- [ ] T048 [US4] Wire move submission in `src/main.py`: on `move_selected(Move)` event, call `moves_api.submit_move()`, update `ApplicationState.current_game` with response snapshot; if game is finished, transition to `GAME_FINISHED`; on error, show toast (do not change state)
- [ ] T049 [US4] Wire resign in `src/main.py`: on resign button click, show confirmation toast/dialog ("Resign? You will lose."); on confirm, call `moves_api.resign()`, transition to `GAME_FINISHED`
- [ ] T050 [US4] Implement game polling during play in `src/state/poller.py` `GamePoller`: extend to call `games_api.get_game()` every `POLLING_INTERVAL` seconds when in `PLAYING_GAME` state; update `ApplicationState.current_game`; trigger re-render; transition to `GAME_FINISHED` if `status == "finished"`; track `last_server_update_time` and `opponent_inactivity_time`
- [ ] T051 [US4] Implement opponent inactivity detection in `src/state/poller.py`: when it is the opponent's turn and no server update received for 30 seconds, show toast "Opponent disconnected. You can resign or wait."; start 5-minute countdown; after 5 minutes with no reconnect, auto-resign the game via `moves_api.resign()`
- [ ] T052 [US4] Create `src/ui/screens/game_over.py`: `GameOverScreen` class; renders game result text (e.g., "White wins by checkmate", "Draw by threefold repetition"); renders "Return to Lobby" button; dispatches `return_to_lobby` event; pixel-art overlay on top of final board position

**Checkpoint**: Full game playable end-to-end; board updates for both players; game-over screen shown; resignation works

---

## Phase 7: User Story 6 – Pixel-Art Visual Experience (Priority: P2)

**Goal**: All screens are rendered in a cohesive pixel-art aesthetic with colour gradations across backgrounds, board, pieces, and UI chrome.

**Independent Test**: Tester unfamiliar with spec identifies art direction as "pixel art with gradation" on all three screens without prompting.

### Implementation

- [ ] T053 [US6] Create placeholder pixel-art colour-gradient background for `UsernameEntryScreen` in `src/assets/sprites/ui/bg_username.png` (64×64 or 128×128 tile); render in `UsernameEntryScreen` using `pygame.transform.scale` to window size; atmospheric dark-muted tones
- [ ] T054 [P] [US6] Create placeholder pixel-art background for `LobbyScreen` in `src/assets/sprites/ui/bg_lobby.png`; render in `LobbyScreen`
- [ ] T055 [P] [US6] Create pixel-art board square sprites: `src/assets/sprites/board/light.png` and `src/assets/sprites/board/dark.png` (64×64 px each); light squares use warm ivory with subtle dithered gradation; dark squares use muted walnut brown with dithering; update `BoardWidget` to render these instead of flat colour fills
- [ ] T056 [P] [US6] Create pixel-art piece sprites for all 12 pieces (6 types × 2 colours) in `src/assets/sprites/pieces/` (64×64 px each, PNG with transparency): `wp.png`, `wn.png`, `wb.png`, `wr.png`, `wq.png`, `wk.png`, `bp.png`, `bn.png`, `bb.png`, `br.png`, `bq.png`, `bk.png`; pixel-art style, hand-crafted, clearly distinguishable at 64×64
- [ ] T057 [P] [US6] Update `src/ui/theme.py` with final palette: 16–32 muted tones (reference: Kathy Rain palette); define `GRADIENT_TOP`, `GRADIENT_BOTTOM` for any programmatic gradients; define `ACCENT_COLOUR` for highlights, selection states, and button hover
- [ ] T058 [P] [US6] Update `PixelArtButton` in `src/ui/components/button.py`: use theme `ACCENT_COLOUR` for hover state; render pixel border (1px inset shadow); add optional icon sprite slot
- [ ] T059 [P] [US6] Update `PixelArtTextInput` in `src/ui/components/text_input.py`: draw pixel-art border using theme; blinking cursor rendered as pixel rect; placeholder text in muted colour
- [ ] T060 [P] [US6] Update `Toast` rendering in `src/ui/components/toast.py`: pixel-art bordered panel; semi-transparent dark background; dismiss animation (fade out over 0.5 seconds)

**Checkpoint**: Tester visual test passes; all screens have consistent pixel-art look; no flat-fill rectangles visible in final build

---

## Phase 8: User Story 5 – Claim a Draw (Priority: P3)

**Goal**: Active player can claim a draw when 50-move rule or threefold repetition condition is eligible.

**Independent Test**: In a position meeting the draw condition, player sees "Claim Draw" button. Clicking it ends the game as a draw on both clients.

### Implementation

- [ ] T061 [US5] Implement `src/adapter/moves_api.py` `claim_draw(http_client, game_id, claim_type) -> dict`; calls `POST /games/{id}/draw`; `claim_type` is `"threefold_repetition"` or `"fifty_move_rule"`; maps server response to `DrawResult(accepted, status, rejection_reason)`
- [ ] T062 [US5] Update `src/ui/screens/board.py`: add "Claim Draw" button visible when it is the player's turn; show draw options dialog (threefold repetition / 50-move rule / both) if multiple conditions may apply; on selection, dispatch `claim_draw(claim_type)` event
- [ ] T063 [US5] Wire draw claim in `src/main.py`: on `claim_draw(type)` event, call `moves_api.claim_draw()`; if accepted, update `ApplicationState.current_game` and transition to `GAME_FINISHED`; if rejected, show toast with `rejection_reason` from server

**Checkpoint**: Draw claim flow works end-to-end; rejected claims show correct server message

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: README, error resilience, performance validation, final integration check

- [ ] T064 Create `README.md` at repository root: project overview, prerequisites, setup (`python3 -m venv venv`, `pip install -r requirements.txt`), run (`python3 -m src.main`), configuration (`.env` table), development commands (tests, type-check, format), project structure tree, build instructions (PyInstaller)
- [ ] T065 [P] Add connection-error screen / retry flow in `src/main.py`: if `create_session()` or `list_games()` raises connection error on startup, show `UsernameEntryScreen` with toast "Server unreachable. Check SERVER_URL and retry."; "Retry" action re-attempts session creation
- [ ] T066 [P] Add `.gitignore` entries: `venv/`, `__pycache__/`, `*.pyc`, `.env`, `dist/`, `build/`, `*.spec`
- [ ] T067 [P] Validate performance characteristics manually: measure time from app launch to lobby (target < 5 sec); measure move-submit-to-board-update latency (target < 500ms); verify idle CPU (target < 2%)
- [ ] T068 Run full integration test: two terminal windows, start server, launch two clients, complete a full game from username entry to checkmate; verify both boards update correctly throughout

---

## Dependencies

### User Story Completion Order

```
Phase 1 (Setup)
    │
    ▼
Phase 2 (Foundation)
    │
    ├──► Phase 3 (US1: Username + Lobby)    ← must complete first (entry point)
    │         │
    │         ├──► Phase 4 (US2: Join Games) [parallel with Phase 5]
    │         │         │
    │         └──► Phase 5 (US3: Create Game) [parallel with Phase 4]
    │                   │
    │              (both complete)
    │                   │
    │                   ▼
    │              Phase 6 (US4: Play Game)  ← requires US2 + US3
    │                   │
    │    ┌──────────────┤
    │    │              │
    │    ▼              ▼
    │  Phase 7 (US6: Visual)   Phase 8 (US5: Draw Claims)
    │    │              │          [parallel with US6]
    │    └──────────────┤
    │                   ▼
    └──────────────► Phase 9 (Polish)
```

### Cross-Story Shared Components

| Component | Used By |
|-----------|---------|
| `http_client.py` | US1, US2, US3, US4, US5 |
| `state_machine.py` | All user stories |
| `toast.py` | All user stories |
| `spinner.py` | US1 (lobby load), US2 (join), US3 (waiting), US4 (move in-flight) |
| `LobbyPoller` | US2 (lobby refresh), US3 (opponent detection) |
| `GamePoller` | US3 (opponent joined), US4 (board updates), US5 (draw eligibility) |
| `theme.py` | US6 (visual), all screens |

---

## Parallel Execution Examples

### After Phase 2 completes, these can run in parallel:

**Track A – US1 (Entry point)**:
T018 → T020 → T021 → T022 → T023 → T024 → T025 → T026 → T027

**Track B – Models (no screen deps)**:
T010 + T011 + T012 (all parallel within Phase 2)

### After US1 completes:

**Track A – US2 (Join)**:
T028 → T029 → T030 → T031 → T032 → T033 → T034

**Track B – US3 (Create)**:
T035 → T036 → T037 → T038 → T039

### After US2 + US3 complete:

**Track A – US4 core logic**:
T040 → T041 → T042 → T045 → T046 → T047 → T048 → T049 → T050

**Track B – US4 components (parallel with A)**:
T043 + T044 (independent component files)

### After US4 completes:

**Track A – US6 (Visual)**:
T053 + T054 + T055 + T056 + T057 + T058 + T059 + T060 (all parallel)

**Track B – US5 (Draw)**:
T061 → T062 → T063

---

## Implementation Strategy

### MVP Scope (Recommended first iteration)

Complete Phase 1 + Phase 2 + Phase 3 (US1) + Phase 6 (US4) first:
- User enters name, session created, lobby appears
- User creates game, waits, second user joins, both play chess to completion
- This delivers the core value proposition

### Incremental Delivery

1. **Sprint 1 (MVP)**: Phases 1–3 + Phase 6 (username entry → board → play game)
2. **Sprint 2**: Phases 4–5 (lobby join + create game flows completed)
3. **Sprint 3**: Phase 7 (pixel-art aesthetic complete)
4. **Sprint 4**: Phase 8 + Polish (draw claims + README + final QA)

---

## Summary

| Phase | User Story | Tasks | Priority |
|-------|-----------|-------|----------|
| Phase 1 | Setup | T001–T007 | - |
| Phase 2 | Foundation | T008–T017 | - |
| Phase 3 | US1: Username + Lobby | T018–T027 | P1 🎯 |
| Phase 4 | US2: Join Games | T028–T034 | P2 |
| Phase 5 | US3: Create Game | T035–T039 | P2 |
| Phase 6 | US4: Play Game | T040–T052 | P1 🎯 |
| Phase 7 | US6: Pixel Art | T053–T060 | P2 |
| Phase 8 | US5: Draw Claims | T061–T063 | P3 |
| Phase 9 | Polish | T064–T068 | - |
| **Total** | | **68 tasks** | |

**Parallel opportunities**: 25+ tasks can be executed in parallel (marked `[P]`)  
**MVP scope**: Phases 1–3 + Phase 6 (T001–T027 + T040–T052) = 40 tasks  
**Independent test criteria**: Each phase has a defined checkpoint verifiable without completing subsequent phases
