# HTTP API Contract: Desktop Client ↔ ajedrez4-server

**Feature**: `001-desktop-chess-client`  
**Date**: 2026-05-02  
**Spec Reference**: [ajedrez4-server/specs/001-chess-game-server/contracts/http-api.md](../../../ajedrez4-server/specs/001-chess-game-server/contracts/http-api.md)

---

## Overview

The desktop client consumes the ajedrez4-server HTTP REST API. This document specifies:
- Which endpoints the client calls
- What the client expects in responses
- How the client handles errors
- Polling strategy and timeouts

---

## Base URL Configuration

```
Server base URL: http://localhost:8080  (default)
Configurable via environment variable:  SERVER_URL=http://chess.example.com:8080
```

---

## Authentication

**Method**: `X-Session-ID` header

```
All endpoints (except POST /sessions) MUST include:
  X-Session-ID: {session_id}
```

Session ID is obtained from `POST /sessions` response and stored in memory.

---

## Endpoints

### 1. POST /sessions — Create Session

**Request**:
```json
POST /sessions HTTP/1.1
Content-Type: application/json

{
  "display_name": "Alice"
}
```

**Response (200 OK)**:
```json
{
  "session_id": "uuid-string",
  "display_name": "Alice",
  "created_at": "2026-05-02T10:30:00Z"
}
```

**Client Behavior**:
- Accept 1–32 printable character usernames (validated locally before sending)
- Store `session_id` in memory for duration of app session
- On error (400, 500): Toast "Failed to create session. Retry?" with retry button

---

### 2. GET /games — List All Games

**Request**:
```
GET /games HTTP/1.1
X-Session-ID: {session_id}
```

**Response (200 OK)**:
```json
{
  "games": [
    {
      "game_id": "game-uuid-1",
      "status": "open",
      "white_player": "Alice",
      "black_player": null,
      "created_at": "2026-05-02T10:30:00Z"
    },
    {
      "game_id": "game-uuid-2",
      "status": "active",
      "white_player": "Bob",
      "black_player": "Charlie",
      "created_at": "2026-05-02T10:25:00Z"
    }
  ]
}
```

**Client Behavior**:
- Call on lobby screen load; refresh every 1–2 seconds (polling)
- Display only `open` and `active` games (filter out `finished` on client)
- Show join button only for `open` games where `white_player` != current username
- On error (401, 500): Toast "Failed to load games. Retry?" with retry button
- On timeout (> 10 seconds): Toast "Server unreachable."

---

### 3. POST /games — Create Game

**Request**:
```
POST /games HTTP/1.1
X-Session-ID: {session_id}
Content-Type: application/json

{}
```

**Response (201 Created)**:
```json
{
  "game_id": "game-uuid",
  "status": "open",
  "white_player": "Alice",
  "black_player": null,
  "created_at": "2026-05-02T10:30:00Z"
}
```

**Client Behavior**:
- Transition to "waiting for opponent" screen
- Show spinner: "Waiting for opponent..."
- Store `game_id`; begin polling (endpoint #2, filtered for this game)
- When polling shows `black_player` is set and `status == "active"`: transition to board screen
- On error (500): Toast "Failed to create game."

---

### 4. GET /games/{id} — Get Game Detail

**Request**:
```
GET /games/{game_id} HTTP/1.1
X-Session-ID: {session_id}
```

**Response (200 OK)** — Active Game:
```json
{
  "game_id": "game-uuid",
  "status": "active",
  "white_player": "Alice",
  "black_player": "Bob",
  "created_at": "2026-05-02T10:30:00Z",
  "active_color": "white",
  "fen": "rnbqkbnr/pppppppp/8/8/4P3/8/PPPP1PPP/RNBQKBNR b KQkq e3 0 1",
  "move_history": ["e2e4"],
  "outcome": null
}
```

**Response (200 OK)** — Finished Game:
```json
{
  "game_id": "game-uuid",
  "status": "finished",
  "white_player": "Alice",
  "black_player": "Bob",
  "created_at": "2026-05-02T10:30:00Z",
  "active_color": null,
  "fen": "...",
  "move_history": ["e2e4", "e7e5", ...],
  "outcome": {
    "result": "white_wins",
    "winner": "white",
    "draw_reason": null
  }
}
```

**Client Behavior**:
- Called during game play (polling every 1–2 seconds)
- Update board display from `fen` and `move_history`
- Check if `active_color` changed to detect opponent's move; reset inactivity timer
- If `status == "finished"`, transition to game-over screen with `outcome`
- Detect opponent inactivity: if no update from server for 30+ seconds and it's the opponent's turn, show toast "Opponent disconnected."
- On error (404): Toast "Game not found. Returning to lobby." + auto-return
- On error (403): Toast "You are not a participant in this game." + auto-return

---

### 5. POST /games/{id}/join — Join Game

**Request**:
```
POST /games/{game_id}/join HTTP/1.1
X-Session-ID: {session_id}
Content-Type: application/json

{}
```

**Response (200 OK)**:
```json
{
  "game_id": "game-uuid",
  "status": "active",
  "white_player": "Alice",
  "black_player": "Bob",
  "created_at": "2026-05-02T10:30:00Z",
  "active_color": "white",
  "fen": "rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1",
  "move_history": [],
  "outcome": null
}
```

**Error (409 Conflict)**:
```json
{
  "error": {
    "code": "game_not_open",
    "message": "Game is no longer open."
  }
}
```

**Error (403 Forbidden)**:
```json
{
  "error": {
    "code": "cannot_join_own_game",
    "message": "Cannot join your own game."
  }
}
```

**Client Behavior**:
- Show loading spinner during request
- On 200: Transition to board screen; begin polling game state (endpoint #4)
- On 409: Toast "Game already started." + return to lobby
- On 403: Toast "Cannot join your own game." + stay in lobby
- On 404: Toast "Game not found." + return to lobby
- On timeout: Toast "Server unreachable. Retry?" with retry button

---

### 6. POST /games/{id}/moves — Submit Move

**Request**:
```
POST /games/{game_id}/moves HTTP/1.1
X-Session-ID: {session_id}
Content-Type: application/json

{
  "notation": "e2e4"
}
```

**Response (200 OK)**:
```json
{
  "accepted": true,
  "game_id": "game-uuid",
  "status": "active",
  "active_color": "black",
  "fen": "rnbqkbnr/pppppppp/8/8/4P3/8/PPPP1PPP/RNBQKBNR b KQkq e3 0 1",
  "move_history": ["e2e4"],
  "outcome": null
}
```

**Error (400 Bad Request)** — Invalid Notation:
```json
{
  "error": {
    "code": "invalid_notation",
    "message": "Invalid move notation."
  }
}
```

**Error (422 Unprocessable Entity)** — Illegal Move:
```json
{
  "error": {
    "code": "illegal_move",
    "message": "Illegal move."
  }
}
```

**Error (403 Forbidden)** — Wrong Turn or Not Participant:
```json
{
  "error": {
    "code": "wrong_turn",
    "message": "It is not your turn."
  }
}
```

**Error (409 Conflict)** — Game Not Active:
```json
{
  "error": {
    "code": "game_not_active",
    "message": "Game is not active."
  }
}
```

**Client Behavior**:
- Disable board interaction while move is in flight (show loading spinner)
- On 200: Update board from response FEN; re-enable board; continue polling
- On 400: Toast "Invalid move syntax. Use e2e4 format." (2–3 seconds)
- On 422: Toast "Illegal move. Check the rules." (2–3 seconds)
- On 403: Toast "It is not your turn." (2–3 seconds)
- On 409: Toast "Game is no longer active. Returning to lobby." + auto-return
- On timeout (> 5 seconds): Toast "Server unreachable. Move cancelled. Retry?" with retry button

---

### 7. POST /games/{id}/resign — Resign

**Request**:
```
POST /games/{game_id}/resign HTTP/1.1
X-Session-ID: {session_id}
Content-Type: application/json

{}
```

**Response (200 OK)**:
```json
{
  "game_id": "game-uuid",
  "status": "finished",
  "outcome": {
    "result": "black_wins",
    "winner": "black",
    "draw_reason": null
  }
}
```

**Client Behavior**:
- Disable board; show spinner: "Processing resignation..."
- On 200: Transition to game-over screen with outcome
- On error (403, 409): Toast "Cannot resign." + stay on board
- On timeout: Toast "Server unreachable. Retry?" with retry button

---

### 8. POST /games/{id}/draw — Claim Draw

**Request**:
```
POST /games/{game_id}/draw HTTP/1.1
X-Session-ID: {session_id}
Content-Type: application/json

{
  "claim_type": "threefold_repetition"
}
```

**Valid claim types**: `"threefold_repetition"`, `"fifty_move_rule"`

**Response (200 OK)** — Draw Accepted:
```json
{
  "accepted": true,
  "game_id": "game-uuid",
  "status": "finished",
  "outcome": {
    "result": "draw",
    "winner": null,
    "draw_reason": "threefold_repetition"
  }
}
```

**Response (200 OK)** — Draw Rejected:
```json
{
  "accepted": false,
  "game_id": "game-uuid",
  "status": "active",
  "rejection_reason": "Draw condition not yet met. (Threefold repetition requires 3 identical positions.)"
}
```

**Error (400 Bad Request)** — Invalid Claim Type:
```json
{
  "error": {
    "code": "invalid_draw_claim_type",
    "message": "Invalid draw claim type."
  }
}
```

**Client Behavior**:
- Show "Claim Draw" button only when `active_color` == current player
- On 200 + `accepted: true`: Transition to game-over screen
- On 200 + `accepted: false`: Toast "Draw claim ineligible: {rejection_reason}" (3–5 seconds); stay on board
- On 400: Toast "Invalid draw claim type." (2–3 seconds)
- On 403, 409: Toast "Cannot claim draw." (2–3 seconds)
- On timeout: Toast "Server unreachable. Retry?" with retry button

---

### 9. DELETE /games/{id} — Cancel Game

**Request**:
```
DELETE /games/{game_id} HTTP/1.1
X-Session-ID: {session_id}
```

**Response (204 No Content)**:
```
HTTP/1.1 204 No Content
```

**Error (403 Forbidden)**:
```json
{
  "error": {
    "code": "cannot_cancel_game",
    "message": "Only the creator can cancel an open game."
  }
}
```

**Client Behavior**:
- Show "Cancel Game" button only on waiting screen if user == white_player
- On 204: Toast "Game cancelled." (auto-dismiss) + return to lobby
- On 403: Toast "Cannot cancel. Game is no longer open or you are not the creator."
- On 404: Toast "Game not found." + return to lobby
- On timeout: Toast "Server unreachable. Retry?" with retry button

---

## Polling Strategy

### Lobby Polling

```
Interval: 1–2 seconds
Endpoint: GET /games
Termination: User leaves lobby (creates or joins game) or closes app
```

### Game Polling

```
Interval: 1–2 seconds
Endpoint: GET /games/{game_id}
Termination: Game finishes OR user resignsOR opponent disconnected + user auto-resigns
```

### Opponent Inactivity Detection

```
Condition: No state update from server for 30+ seconds while it's opponent's turn
Action: Toast "Opponent disconnected. You can resign or wait."
Timeout: 5 minutes; auto-resign if opponent doesn't rejoin
```

---

## Error Handling & Toast Notifications

### Error Classification

| HTTP Code | Error Code | Toast Message | Action |
|-----------|-----------|---|---|
| 400 | invalid_notation | "Invalid move syntax. Use e2e4 format." | Dismiss (auto) |
| 400 | invalid_draw_claim_type | "Invalid draw claim type." | Dismiss (auto) |
| 400 | invalid_display_name | "Name must be 1–32 characters." | Dismiss (auto) |
| 401 | missing_session_header | "Session expired. Return to username screen." | User clicks button |
| 401 | session_not_found | "Session not found." | User clicks button → return to entry |
| 403 | cannot_join_own_game | "Cannot join your own game." | Dismiss (auto) |
| 403 | cannot_cancel_game | "Cannot cancel. You are not the creator." | Dismiss (auto) |
| 403 | not_a_participant | "You are not in this game." | User clicks button → return to lobby |
| 403 | wrong_turn | "It is not your turn." | Dismiss (auto) |
| 404 | game_not_found | "Game not found. Returning to lobby." | Auto-dismiss (2s) → return |
| 409 | game_not_open | "Game is no longer open." | User clicks button → return to lobby |
| 409 | game_not_active | "Game is not active." | User clicks button → return to lobby |
| 422 | illegal_move | "Illegal move. Check the rules." | Dismiss (auto) |
| 500 | server_error | "Server error. Retry?" | User clicks "Retry" |
| timeout | connection_timeout | "Server unreachable. Retry?" | User clicks "Retry" |

### Toast Notification Pattern

```python
@dataclass
class Toast:
    message: str
    duration_sec: int  # 0 = manual dismiss, 2–5 = auto-dismiss
    action_button: Optional[str] = None  # "Retry", "Return", etc.
    action_callback: Optional[Callable] = None
```

---

## Timeouts

| Operation | Timeout | Behavior on Timeout |
|-----------|---------|---|
| Any HTTP request | 10 seconds | Toast "Server unreachable. Retry?" |
| Polling (game in progress) | 10 seconds per request | Treat as missed update; continue polling |
| Opponent inactivity | 30 seconds of no server update | Toast "Opponent disconnected." |
| Opponent reconnection window | 5 minutes | Auto-resign if no update; confirm with toast |

---

## Next Steps

1. **Quickstart.md**: Build + run, environment setup, development workflow.
2. **Implementation tasks**: Tasks generated by `/speckit.tasks` will reference these contracts.
