# Data Model & State Machine: Desktop Chess Client

**Feature**: `001-desktop-chess-client`  
**Date**: 2026-05-02  
**Status**: Phase 1 Design  
**Input**: Research.md (Python + Pygame decision)

---

## Application State Machine

### States

```
                    ┌─────────────────┐
                    │   IDLE / INIT   │
                    └────────┬────────┘
                             │ startup()
                             ▼
                    ┌─────────────────┐
                    │  USERNAME_ENTRY │
    ┌──────────────►│  (screen shown) │◄──┐
    │               └────────┬────────┘    │
    │ cancel_game()          │             │
    │ or timeout             │ submit_username()
    │                        ▼             │
    │               ┌─────────────────┐    │
    │               │ FETCHING_LOBBY  │    │
    │               └────────┬────────┘    │
    │                        │             │ session_error()
    │                        ▼ OK          │
    │               ┌─────────────────┐    │
    └───────────────│    IN_LOBBY     │────┘
                    │  (game list +   │
                    │   buttons)      │
                    └─┬──────────┬────┘
        ┌──────────────┘          └──────────────┐
        │ create_game()                  join_game()
        ▼                                ▼
   ┌─────────────────┐              ┌─────────────────┐
   │ WAITING_FOR_OPP │              │ FETCHING_BOARD  │
   │ (spinner shown) │              └────────┬────────┘
   └────┬──────┬────┘                        │
        │      │                              ▼ OK
        │      └──────────────┐         ┌─────────────────┐
        │ opponent_joined()   └────────►│  PLAYING_GAME   │
        ▼                               │ (board shown)   │
   ┌─────────────────┐                  └────┬────────┬───┘
   │  PLAYING_GAME   │                       │        │
   │ (board shown)   │                       │        │
   └────┬────────┬───┘                       │        │
        │        │                          │        │
        │        ├─ move_submitted() ──────►│        │
        │        │                          │        │
        │        ├─ resign_clicked() ───────┼────┐   │
        │        │                          │    │   │
        │        └─ draw_claimed() ─────────┼────┤   │
        │                                   │    │   │
        ▼                                   ▼    │   │
   ┌─────────────────┐               ┌─────────────┴──┐
   │ GAME_FINISHED   │◄──────────────│ (via polling)  │
   │ (result shown)  │               │ + update board │
   └────┬────────────┘               └────────────────┘
        │
        │ return_to_lobby()
        ▼
   ┌─────────────────┐
   │    IN_LOBBY     │
   │ (game list +    │
   │  buttons)       │
   └─────────────────┘
```

### State Definitions

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional

class AppState(Enum):
    IDLE = "idle"
    USERNAME_ENTRY = "username_entry"
    FETCHING_LOBBY = "fetching_lobby"
    IN_LOBBY = "in_lobby"
    WAITING_FOR_OPP = "waiting_for_opp"
    FETCHING_BOARD = "fetching_board"
    PLAYING_GAME = "playing_game"
    OPPONENT_DISCONNECTED = "opponent_disconnected"
    GAME_FINISHED = "game_finished"

@dataclass
class ApplicationState:
    """The complete application state at any moment."""
    
    current_state: AppState
    session_id: Optional[str] = None
    username: Optional[str] = None
    game_id: Optional[str] = None
    current_game: Optional['GameSnapshot'] = None
    
    # UI state
    selected_piece: Optional[tuple[int, int]] = None  # (row, col) if piece selected
    valid_moves: list[tuple[int, int]] = None  # Destination squares for selected piece
    
    # Polling
    last_server_update_time: float = 0.0  # Timestamp of last successful poll
    opponent_inactivity_time: float = 0.0  # Track opponent's move timestamp
    
    # Error state
    last_error: Optional[str] = None  # Human-readable error message for toast
    error_action: Optional[str] = None  # "retry", "return_to_lobby", or None
```

---

## Domain Types

### Session

```python
@dataclass
class Session:
    """User session created on server."""
    session_id: str          # UUID from server
    username: str            # 1-32 printable chars
    created_at: str          # ISO timestamp
```

### Game

```python
from enum import Enum

class GameStatus(Enum):
    OPEN = "open"            # Waiting for 2nd player
    ACTIVE = "active"        # In progress
    FINISHED = "finished"    # Ended (checkmate, resignation, draw)

class GameResult(Enum):
    WHITE_WINS = "white_wins"
    BLACK_WINS = "black_wins"
    DRAW = "draw"
    IN_PROGRESS = "in_progress"

@dataclass
class GameOutcome:
    """Result of a finished game."""
    result: GameResult
    winner: Optional[str]      # "white", "black", or None (for draw)
    draw_reason: Optional[str] # "threefold", "fifty_move", "stalemate", etc.

@dataclass
class GameSnapshot:
    """Snapshot of game state retrieved from server."""
    game_id: str
    status: GameStatus
    white_player: Optional[str]
    black_player: Optional[str]
    fen: Optional[str]              # None if game is open (no board yet)
    move_history: list[str]         # UCI notation: ["e2e4", "e7e5", ...]
    active_color: Optional[str]     # "white" or "black"
    outcome: Optional[GameOutcome]  # None if game is active
    created_at: str                 # ISO timestamp
```

### Move & Promotion

```python
from enum import Enum

class Piece(Enum):
    QUEEN = "q"
    ROOK = "r"
    BISHOP = "b"
    KNIGHT = "n"

@dataclass
class Move:
    """A chess move entered by the user."""
    from_square: str      # e.g., "e2" (algebraic notation)
    to_square: str        # e.g., "e4"
    promotion: Optional[Piece] = None  # For pawn promotion
    
    def to_uci(self) -> str:
        """Convert to UCI notation expected by server."""
        uci = f"{self.from_square}{self.to_square}"
        if self.promotion:
            uci += self.promotion.value
        return uci
```

---

## UI-Specific Types

### Screen Events

```python
@dataclass
class ClickEvent:
    """User clicked on the board or a button."""
    x: int
    y: int
    button: int  # 1 = left, 3 = right

@dataclass
class KeyEvent:
    """User pressed a key."""
    key: str

@dataclass
class UIEvent:
    """Union of all user interactions."""
    type: str  # "click", "key", "hover"
    click: Optional[ClickEvent] = None
    key: Optional[KeyEvent] = None
```

### Piece Position & Board Logic

```python
from typing import Optional

@dataclass
class SquareCoord:
    """A square on the chess board in pixel coordinates."""
    x: int
    y: int
    row: int  # 0-7 (bottom to top for white; top to bottom for black)
    col: int  # 0-7 (left to right)

class BoardOrientation(Enum):
    WHITE = "white"  # White at bottom
    BLACK = "black"  # Black at bottom (flipped 180°)

def screen_to_square(screen_x: int, screen_y: int, orientation: BoardOrientation) -> Optional[SquareCoord]:
    """Convert pixel coordinates to board square."""
    # Pseudocode:
    # - Calculate which cell was clicked (board_x = screen_x // SQUARE_SIZE, etc.)
    # - If orientation is BLACK, flip the coordinates
    # - Return SquareCoord or None if outside board
    pass

def fen_to_board(fen: str) -> list[list[Optional[str]]]:
    """Parse FEN string into 8x8 board representation."""
    # Extract board portion of FEN (before first space)
    # Split by "/" to get ranks
    # Expand numbers to empty squares
    # Return 8x8 array where None = empty, "wp" = white pawn, "bk" = black king, etc.
    pass

def get_valid_moves(fen: str, from_square: str) -> list[str]:
    """Query server or use local library for legal moves from a square."""
    # For now: rely on server validation (send move, get 400 if illegal)
    # Could integrate python-chess library for client-side validation
    pass
```

---

## Server API Contract Types

### HTTP Responses

```python
@dataclass
class SessionResponse:
    session_id: str
    display_name: str
    created_at: str

@dataclass
class GameSummaryResponse:
    game_id: str
    status: str  # "open", "active", "finished"
    white_player: Optional[str]
    black_player: Optional[str]
    created_at: str

@dataclass
class GameDetailResponse:
    game_id: str
    status: str
    white_player: Optional[str]
    black_player: Optional[str]
    created_at: str
    active_color: Optional[str]       # "white", "black"
    fen: Optional[str]
    move_history: list[str]
    outcome: Optional[dict]           # {"result": "...", "winner": "...", ...}

@dataclass
class MoveResponse:
    accepted: bool
    game_id: str
    status: str
    active_color: Optional[str]
    fen: str
    move_history: list[str]
    outcome: Optional[dict]

@dataclass
class ErrorResponse:
    error: dict  # {"code": "...", "message": "..."}
```

---

## State Transitions & Event Handlers

### Example: Joining a Game

```python
async def join_game_handler(app_state: ApplicationState, game_id: str) -> ApplicationState:
    """Handle user clicking 'Join' on a game."""
    
    # Transition to fetching state
    app_state.current_state = AppState.FETCHING_BOARD
    yield app_state  # Re-render (show spinner)
    
    try:
        # Call server API
        response = await http_client.join_game(game_id, app_state.session_id)
        game = GameSnapshot.from_response(response)
        
        # Update application state
        app_state.game_id = game_id
        app_state.current_game = game
        app_state.current_state = AppState.PLAYING_GAME
        app_state.last_server_update_time = time.time()
        
    except HTTPError as e:
        app_state.last_error = translate_error(e)  # "Game already started" etc.
        app_state.error_action = "return_to_lobby"
        app_state.current_state = AppState.IN_LOBBY
    
    return app_state
```

### Example: Polling Loop

```python
async def polling_loop(app_state: ApplicationState, interval_sec: float = 1.5) -> None:
    """Continuously fetch game state from server."""
    
    while app_state.current_state in [AppState.PLAYING_GAME, AppState.OPPONENT_DISCONNECTED]:
        try:
            response = await http_client.get_game(app_state.game_id, app_state.session_id)
            game = GameSnapshot.from_response(response)
            
            # Detect opponent inactivity
            if game.active_color != app_state.current_game.active_color:
                # Opponent made a move; reset inactivity timer
                app_state.opponent_inactivity_time = time.time()
            
            # Check if opponent is inactive (30+ seconds with no update from them)
            time_since_opponent_move = time.time() - app_state.opponent_inactivity_time
            if time_since_opponent_move > 30 and game.status == "active":
                app_state.current_state = AppState.OPPONENT_DISCONNECTED
                app_state.last_error = "Opponent disconnected. You can resign or wait."
                app_state.error_action = "resign_or_wait"
            
            # Check for game completion
            if game.status == "finished":
                app_state.current_state = AppState.GAME_FINISHED
            
            # Update board
            app_state.current_game = game
            app_state.last_server_update_time = time.time()
            
        except HTTPError as e:
            app_state.last_error = translate_error(e)
            app_state.current_state = AppState.IN_LOBBY  # or OPPONENT_DISCONNECTED
        
        await asyncio.sleep(interval_sec)
```

---

## Key Invariants

1. **Session ID**: Once set, persists until user returns to username entry screen.
2. **Game ownership**: If `white_player == username`, user can cancel an open game; otherwise cannot.
3. **Turn validation**: Client MUST not allow piece selection/moves unless `active_color` == user's color.
4. **Board orientation**: If user is black, board MUST be flipped 180° (rotation in screen coordinates or FEN rank reversal).
5. **Polling synchrony**: Only one polling thread/task active at a time; new game transitions cancel previous poll.
6. **Error transience**: Toast errors auto-dismiss; critical errors require user action (button click).

---

## Next Steps (Phase 1 continuation)

1. **Contracts/server-api.md**: Document HTTP endpoints, polling interval, error codes, timeouts.
2. **Quickstart.md**: Build + run instructions, environment setup, development workflow.
3. **Update .github/copilot-instructions.md**: Add plan.md reference.
