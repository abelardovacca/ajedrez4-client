# Feature Specification: Desktop Chess Client

**Feature Branch**: `001-desktop-chess-client`  
**Created**: 2026-05-01  
**Status**: Draft  
**Input**: User description: "a desktop client for the chess game, that relies on the ajedrez4-server for its backend functionality. When a user opens the client he is prompted to enter a username (there is no authentication), then gets taken to a screen with the list of games, where he can create a new game or join an existing one that is still missing one player. Once a game has two players it starts automatically. The color pieces are assigned randomly, the black pieces player should have his board flipped. The client should use pixel-art with beautiful gradation, in the style of Kathy Rain."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Enter Username and Access Lobby (Priority: P1)

A new user launches the desktop client and is presented with a username entry screen. After entering a name, they are taken to the games lobby.

**Why this priority**: This is the entry point of the entire application. Without it, no other functionality is accessible.

**Independent Test**: Launch the application, enter a username, and confirm the lobby screen appears with a game list. Delivers a usable shell of the application even before game play is functional.

**Acceptance Scenarios**:

1. **Given** the app is launched, **When** the username screen appears, **Then** the user sees a text input and a confirm button.
2. **Given** the username field is empty, **When** the user attempts to confirm, **Then** confirmation is blocked and an error hint is shown.
3. **Given** a valid username is entered (1–32 printable characters), **When** the user confirms, **Then** the lobby screen is displayed.
4. **Given** a username longer than 32 characters is entered, **When** the user attempts to confirm, **Then** the input is rejected with a clear message.

---

### User Story 2 - View and Join Open Games from Lobby (Priority: P2)

From the lobby, the user sees a list of open games (those waiting for a second player). The user can join any open game with one click.

**Why this priority**: Joining an existing game is the primary multi-player interaction; without it, two players cannot play together.

**Independent Test**: With one game already open on the server, a second user launches the client, enters a username, sees the game listed, and joins it — confirming they reach the game board screen.

**Acceptance Scenarios**:

1. **Given** the lobby is displayed, **When** games exist on the server, **Then** each game shows the creator's name and its status.
2. **Given** a game has status "open" (missing one player), **When** the user clicks Join, **Then** the client sends a join request and transitions to the game board screen.
3. **Given** a game is already active (two players), **When** it appears in the list, **Then** no join option is presented for that game.
4. **Given** the lobby is displayed, **When** no games exist, **Then** an empty-state message is shown.

---

### User Story 3 - Create a New Game (Priority: P2)

From the lobby, a user can create a new game and wait for an opponent to join.

**Why this priority**: Creation is the complement to joining; both must exist for two players to meet.

**Independent Test**: A single user creates a game and sees the waiting screen showing they are waiting for an opponent. The game appears in another user's lobby list.

**Acceptance Scenarios**:

1. **Given** the lobby is displayed, **When** the user clicks "New Game", **Then** a new game is created on the server and the user transitions to a waiting screen.
2. **Given** the user is on the waiting screen, **When** a second player joins from their client, **Then** both clients automatically transition to the game board screen without any manual action.
3. **Given** the user is waiting, **When** they choose to cancel, **Then** the game is deleted and the user returns to the lobby.

---

### User Story 4 - Play a Chess Game (Priority: P1)

Once two players are in a game, they play chess on the board screen. The correct side moves first, pieces are moved by click-and-click or drag-and-drop, and the board is flipped for the black player.

**Why this priority**: The chess board is the core value of the application.

**Independent Test**: Two users in an active game can alternate moves until the game ends (checkmate, resignation, or draw claim). The board state updates in real time on both clients.

**Acceptance Scenarios**:

1. **Given** a game starts, **When** a player views their board, **Then** the board is oriented so their pieces are at the bottom (black's board is flipped 180°).
2. **Given** it is white's turn, **When** white clicks a piece then a valid destination, **Then** the move is submitted and the board updates to reflect the new state.
3. **Given** it is white's turn, **When** black attempts to move, **Then** the action is rejected locally with a visual indication.
4. **Given** a move is submitted, **When** it is accepted by the server, **Then** the opponent's board also updates to show the new position.
5. **Given** a player's king is in checkmate, **When** the final move is made, **Then** both clients show a game-over screen with the result.
6. **Given** a player wishes to resign, **When** they activate the resign option, **Then** the game ends with the opponent winning, and both screens update.

---

### User Story 6 - Immersive Pixel-Art Visual Experience (Priority: P2)

Every screen of the client is rendered in a cohesive pixel-art aesthetic inspired by point-and-click adventure games like *Kathy Rain*: hand-crafted sprites with rich colour gradations, atmospheric backgrounds, and expressive UI chrome that feels like a painted scene rather than a standard application window.

**Why this priority**: Visual identity is a core product differentiator. A bare-functional board with no aesthetic coherence would not meet the product vision, making this higher priority than optional chess rules (draw claims).

**Independent Test**: A tester unfamiliar with the spec can identify the art direction as "pixel art with gradation" on all three screens (username entry, lobby, game board) without being told the style intent.

**Acceptance Scenarios**:

1. **Given** any screen of the application is displayed, **When** a user looks at it, **Then** all UI elements (backgrounds, buttons, text boxes, piece sprites, board squares) are rendered in a consistent pixel-art style.
2. **Given** the game board is shown, **When** the user observes the board and pieces, **Then** pieces are pixel-art sprites and the board features colour gradations (e.g., subtle lighting, atmospheric depth) rather than flat solid fills.
3. **Given** any UI element receives focus or is hovered, **When** the user interacts with it, **Then** the interaction feedback (highlight, selection) is consistent with the pixel-art visual language.
4. **Given** the username screen is displayed, **When** a user first launches the app, **Then** the screen presents an atmospheric pixel-art background that establishes the game's mood.

---

### User Story 5 - Claim a Draw (Priority: P3)

A player whose turn it is can claim a draw if the 50-move rule or threefold repetition condition is met.

**Why this priority**: Draw claims are a chess rule completion item; the game is playable without them but incomplete.

**Independent Test**: In a position meeting the 50-move or threefold condition, the claiming player sees a draw claim option. Activating it ends the game as a draw.

**Acceptance Scenarios**:

1. **Given** it is the player's turn and a draw condition is eligible, **When** the player claims a draw, **Then** the game ends as a draw and both clients show the result.
2. **Given** no draw condition is eligible, **When** the player attempts to claim a draw, **Then** the server rejects it and the player is informed.

---

### Edge Cases

- What happens if the server is unreachable when the client starts? → Show a connection error and allow retry.
- What happens if the game the user is in is cancelled by the opponent (before joining)? → Redirect back to the lobby with a notification.
- What happens if the opponent disconnects mid-game? → The client detects opponent inactivity via polling timeout and shows an explicit toast: "Opponent disconnected. You can resign or wait." The board remains visible; the player can resign or wait up to 5 minutes for reconnection. If the opponent does not reconnect within 5 minutes, the game is automatically resigned (opponent loss).
- What if two users try to join the same open game simultaneously? → Only one succeeds; the other sees a "game already started" message and is returned to the lobby.
- What if a pawn reaches the last rank? → A promotion dialog appears, letting the player choose the piece (queen, rook, bishop, knight).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The client MUST display a username entry screen on first launch and whenever no session is active.
- **FR-002**: The client MUST validate that the username is between 1 and 32 printable characters before accepting it.
- **FR-003**: The client MUST create a server session using the entered username and store the session ID for subsequent requests.
- **FR-004**: After session creation, the client MUST navigate to the lobby screen automatically.
- **FR-005**: The lobby screen MUST display the current list of games fetched from the server, including creator name and status.
- **FR-006**: The lobby MUST distinguish open games (joinable) from active/finished games and only show a Join action for open games the current user did not create.
- **FR-007**: The client MUST allow the user to create a new game, which transitions to a waiting-for-opponent screen.
- **FR-008**: The waiting screen MUST poll or refresh the game state and automatically transition to the game board when a second player joins.
- **FR-009**: The client MUST allow the user to cancel their own open game from the waiting screen, returning them to the lobby.
- **FR-010**: The game board MUST render all pieces in their correct positions as returned by the server's FEN string.
- **FR-011**: The board MUST be oriented with the current user's pieces at the bottom; black's board MUST be flipped 180°.
- **FR-012**: The client MUST only allow the active player to interact with pieces when it is their turn.
- **FR-013**: Move input MUST be via click-to-select then click-to-place (and optionally drag-and-drop).
- **FR-014**: When a pawn reaches the promotion rank, the client MUST present a piece-selection dialog before submitting the move.
- **FR-015**: After each move, the client MUST refresh the game state from the server so the opponent's board updates.
- **FR-016**: The client MUST provide a Resign button that ends the game with the opponent winning.
- **FR-017**: When a draw condition is eligible (communicated by the server), the client MUST surface a Claim Draw option for the active player.
- **FR-018**: The game board MUST display the game result (winner, draw reason) when the game reaches a finished state.
- **FR-019**: The client MUST handle server error responses gracefully, displaying a human-readable message without crashing.
- **FR-020**: The entire client interface MUST be rendered in a pixel-art visual style with colour gradations, consistent across all screens (username entry, lobby, waiting screen, game board, game-over overlay).
- **FR-021**: Chess piece sprites MUST be pixel-art illustrations that are clearly distinguishable by type and colour at the board's display size.
- **FR-022**: UI elements (buttons, inputs, dialogs, backgrounds) MUST use a cohesive pixel-art theme with atmospheric depth achieved through colour gradations rather than flat fills.
- **FR-024**: During async operations (joining, waiting for opponent, loading board), the client MUST display an animated pixel-art spinner or throbber with descriptive text so the user perceives the app as responsive.
- **FR-025**: Error and warning messages MUST be displayed as non-blocking toast notifications that auto-dismiss after 3–5 seconds; critical errors MUST offer an action (retry or return-to-lobby) in addition to the dismissible toast.
- **FR-026**: The client MUST detect opponent inactivity by tracking the most recent move timestamp from polling. If no update is received from the opponent for 30+ seconds during their turn, the client MUST display a toast: "Opponent disconnected. You can resign or wait." The board MUST remain interactive; the current player can resign or claim victory if the disconnect persists for 5 minutes.

### Key Entities

- **Session**: The user's identity for the current run; holds a display name and a server-issued session ID.
- **Game**: A chess match with two player slots, a status (open / active / finished), and a board state (FEN + move history).
- **Board**: A visual representation of the chess position derived from the game's FEN; oriented per the player's color.
- **Move**: A user action selecting a source and destination square; submitted to the server in UCI notation.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can go from launching the app to making their first move in under 60 seconds.
- **SC-002**: Two users on the same local network can complete a full game (from lobby to checkmate) without encountering an unhandled error.
- **SC-003**: The board state shown to each player is consistent with the server state within one polling cycle after any move.
- **SC-004**: 100% of chess rules enforced by the server are surfaced to the user as clear feedback (invalid move, wrong turn, game over) rather than silent failures.
- **SC-005**: The black player's board is always displayed flipped relative to white's board.
- **SC-006**: A user unfamiliar with the spec can identify all three primary screens as sharing a single coherent pixel-art visual style on first viewing.

## Clarifications

### Session 2026-05-01

- Q: Target desktop platform? → A: Windows + macOS + Linux (cross-platform)
- Q: Polling interval? → A: 1–2 seconds (standard turn-based game)
- Q: Async feedback during waits? → A: Animated spinner + "Waiting for opponent..." text
- Q: Error display modality? → A: Toast notifications (non-blocking, auto-dismiss in 3–5 seconds)
- Q: Opponent disconnection handling? → A: Explicit toast: "Opponent disconnected. You can resign or wait."

## Assumptions

- The ajedrez4-server is running and reachable; its base URL is configured in the client (e.g., via a config file or environment variable defaulting to `http://localhost:8080`).
- There is no persistent identity: each time the user launches the app and enters a name, a new session is created on the server. No login, password, or account storage is required.
- Color assignment (white/black) is determined by the server: the game creator is white, the joiner is black. The description states "randomly assigned" — this is interpreted as: from the user's perspective the color is unpredictable since they don't choose whether to create or join.
- The lobby does not show finished games; only open and active games are listed.
- Polling is acceptable for keeping the lobby and waiting screen up-to-date with a 1–2 second interval; a real-time push mechanism is out of scope for v1.
- Pawn promotion defaults to queen if the user does not interact with the dialog within a short timeout (or the dialog always blocks until a choice is made).
- Mobile support is out of scope; the client targets desktop operating systems (Windows, macOS, Linux) only, with a single-codebase cross-platform implementation preferred.
- The ajedrez4-server's HTTP API contract (from ajedrez4-server specs) is the authoritative interface; no changes to the server are in scope.
- The pixel-art visual style is inspired by *Kathy Rain* (Raw Fury, 2016): low-resolution hand-drawn sprites, rich atmospheric colour gradations, and a moody point-and-click adventure aesthetic. This is a directional reference, not a requirement to reproduce copyrighted assets.
- Custom pixel-art assets (piece sprites, backgrounds, UI chrome) are to be created as part of this feature; no third-party asset pack is assumed.
- Window size and render resolution are unspecified in v1; target a 1024×768 minimum resolution (typical for point-and-click adventure games of the *Kathy Rain* era).
- Async operations (joining game, loading board, waiting for opponent) MUST provide clear visual feedback: an animated spinner or throbber paired with descriptive text (e.g., "Waiting for opponent...", "Loading game board...") to signal that the app is responsive and working.
- Error messages MUST be presented as non-blocking toast notifications that auto-dismiss after 3–5 seconds, allowing the user to continue interacting with the UI. Critical errors (e.g., server unreachable, session invalid) MUST also offer a retry or return-to-lobby action.
- Opponent disconnection is inferred from polling timeout: if no state update is received for 30+ seconds, the client displays an explicit toast notification and allows the connected player to resign or wait. A 5-minute inactivity limit triggers automatic resignation (opponent loss).
