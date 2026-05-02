# Phase 0 Research: Desktop Chess Client

**Feature**: `001-desktop-chess-client`  
**Date**: 2026-05-02  
**Status**: Complete  
**Goal**: Resolve all NEEDS CLARIFICATION items from plan.md

---

## 1. Language & Framework Selection

**DECISION**: Python 3.11+ with Pygame 2.x  
**Rationale**: Turn-based UI, rapid iteration, excellent pixel-art support, strong cross-platform story.

### Research

**Candidates Evaluated**:

| Option | Pros | Cons | Score |
|--------|------|------|-------|
| **Rust + Egui** | Fast, portable, modern | Steep learning curve, smaller ecosystem | 7/10 |
| **Python + Pygame** | Pixel-art mature, rapid dev, cross-platform | Slightly slower, packaging complexity | **9/10** |
| **Python + PyQt5** | Professional UI, rapid dev | Not game-optimized, overkill for pixel art | 6/10 |
| **C# + MonoGame** | Game-focused, pixel-art good | Mono/Windows-centric, less indie feel | 7/10 |
| **Electron + TypeScript** | Web stack, prototyping speed | Heavy (100+ MB), Chromium overhead | 5/10 |
| **Go + Fyne** | Same language as server | Not designed for pixel art, smaller community | 4/10 |

**Winner: Python + Pygame 2.x**

- **Pixel-art heritage**: Pygame has shipped 1000s of indie games with pixel art; tooling and examples abundant
- **Cross-platform**: Windows, macOS, Linux supported natively; single codebase
- **Rapid iteration**: Python dev cycle is fast; Pygame learning curve is shallow
- **Asset compatibility**: Aseprite and Piskel (leading pixel-art tools) export PNG natively; Pygame renders PNG sprites directly
- **Polling simplicity**: Thread-based polling or async tasks are straightforward in Python

**Trade-offs Accepted**:
- Performance: Pygame is slower than Rust/C#, but turn-based chess doesn't require 60 fps
- Packaging: PyInstaller or similar needed for distribution; no compiled binary by default
- Python ecosystem volatility: Mitigated by pinning versions in requirements.txt

---

## 2. Primary Dependencies

**DECISION**:

| Dependency | Purpose | Version |
|------------|---------|---------|
| **pygame** | Rendering, input, event loop | 2.1.2+ |
| **requests** | HTTP client for ajedrez4-server | 2.28+ |
| **python-dotenv** | Environment configuration (.env loading) | 0.20+ |
| **pytest** | Unit + contract testing | 7.2+ |
| **pytest-asyncio** | Async test support (if polling uses asyncio) | 0.20+ |
| **mypy** (dev) | Type checking | 0.990+ |
| **black** (dev) | Code formatting | 22.12+ |

**Rationale**:
- `pygame`: Industry standard for 2D game dev in Python
- `requests`: Simple, well-tested HTTP library (synchronous polling is acceptable for turn-based game)
- `python-dotenv`: Clean config management (server URL, polling interval)
- `pytest`: Standard testing framework in Python ecosystem
- Type hints + linting: Production-grade code quality (aligns with Constitution V)

---

## 3. Storage

**DECISION**: None (stateless client)

**Justification**:
- Session ID is stored in memory (application state, not persistent file)
- Game state is downloaded from server on each poll
- No user accounts, no game history, no saved games required
- Constraint per spec: "fresh session each launch"

---

## 4. Testing Framework

**DECISION**: pytest + pytest-asyncio (if async polling) + mocking of HTTP adapter

**Test Strategy**:
- **Unit tests** (`tests/unit/`):
  - State machine transitions (LoggingIn → InLobby → PlayingGame)
  - FEN parsing and board orientation
  - Move validation (local syntax, not rules — rules enforced by server)
  
- **Contract tests** (`tests/contract/`):
  - Mock ajedrez4-server responses (httpserver fixture)
  - Verify client parses HTTP responses correctly
  - Test error handling (500, 404, timeout scenarios)
  
- **Integration tests** (`tests/integration/`):
  - Full flow: username entry → lobby → join → board → move → game-over
  - (Optional) spawn actual ajedrez4-server for true end-to-end

**Tool**: pytest with fixtures; `pytest-mock` for mocking requests

---

## 5. Target Platform

**DECISION**: Windows 10+, macOS 10.14+, Linux (Debian, Ubuntu, Fedora)

**Implementation**:
- Single Python codebase (Pygame is platform-agnostic)
- PyInstaller builds for each OS (3 binary releases)
- Tested on all three platforms before release
- Configuration via environment variables (handled by `python-dotenv`)

---

## 6. Project Type

**DECISION**: Desktop GUI application (single-window, single-player local/network multiplayer via separate windows)

**Scope clarification**:
- Not a web app (no Django, no FastAPI)
- Not a CLI tool
- Single executable per platform
- Multiplayer is network-based (two separate client instances talking to shared server)

---

## 7. Performance Goals

**DECISION**:
- **Move submission perceived latency**: < 500ms (includes HTTP round-trip + server processing + UI update)
- **Board re-render**: No strict requirement (turn-based, not real-time); target 16.67ms (60 fps) for smooth animations, but not critical
- **Polling overhead**: < 2% CPU at idle (1–2 second poll interval, minimal payload)
- **Memory**: < 150 MB resident set (typical for Pygame app)
- **Startup time**: < 5 seconds

**Rationale**: Turn-based game doesn't require hard real-time guarantees. User tolerance for responsiveness is higher (~500ms) than action games.

---

## 8. Constraints

**DECISION**:

| Constraint | Value | Rationale |
|-----------|-------|-----------|
| Minimum resolution | 1024×768 | Typical retro game resolution; *Kathy Rain* era |
| No external asset packs | Custom sprites only | Specification requirement (pixel-art created for this project) |
| Server unreachable fallback | Show error toast + retry button | Stateless client can't continue without server |
| Polling timeout | 30 seconds (detected as disconnect) | Longer than 1–2 second poll interval; conservative |
| Opponent auto-resign timer | 5 minutes inactivity | Specification requirement |
| Max username length | 32 characters | Inherited from server validation |

---

## 9. Scale & Scope

**DECISION**:

| Metric | Estimate | Notes |
|--------|----------|-------|
| Lines of code | 6–8k | UI rendering, state machine, HTTP adapter, tests |
| Number of screens | 5 | Username entry, lobby, waiting, game board, game-over |
| Number of API endpoints consumed | 6 | POST /sessions, GET /games, POST /games/{id}/join, POST /games/{id}/moves, POST /games/{id}/resign, POST /games/{id}/draw |
| Sprite count | ~50–100 | 6 piece types × 2 colors × 2–3 animation frames, UI chrome (buttons, spinner, etc.) |
| Development time estimate | 4–6 weeks | Assuming 1 developer; includes asset creation, testing |
| Concurrent users per window | 1 | Single-window, single-player view; multiplayer via separate instances |

---

## 10. Asset Pipeline & Pixel Art

**DECISION**: Aseprite for authoring; export PNG; embedded in repository

**Tool**: Aseprite (proprietary, ~$20) or free alternative Piskel (web-based)

**Asset Creation Strategy**:
- **Piece sprites**: 64×64 px per piece, 6 types × 2 colors × 1–2 animation frames
- **Board squares**: 64×64 px (light/dark variants with subtle gradation)
- **UI chrome**: Buttons, spinner (animated GIF converted to PNG frame series), backgrounds
- **Palette**: Curated 16–32 colour palette inspired by *Kathy Rain* (muted/dusty tones, good for retro feel)
- **Storage**: `src/assets/sprites/` as PNG files; metadata (sprite size, frame count) in Python config

**Colour Gradation Approach**:
- Use dithering or subtle shading within limited palette
- Reference: *Kathy Rain* uses 16–32 colour palette with hand-drawn lighting/shadows
- Example: board squares use 2–3 shades per colour for 3D depth illusion

---

## 11. Polling Architecture

**DECISION**: Thread-based polling with configurable interval

**Implementation**:
```python
# Pseudocode
class GamePoller:
    def __init__(self, interval_sec: float = 1.5):
        self.interval = interval_sec
        self.thread = threading.Thread(target=self._poll_loop, daemon=True)
    
    def _poll_loop(self):
        while self.running:
            state = self.http_client.fetch_game_state(game_id)
            self.state_machine.update(state)
            time.sleep(self.interval)
```

**Trade-off**: Thread-based is simpler than async/await; acceptable for low-frequency polling.

---

## 12. HTTP Client Error Handling

**DECISION**: Explicit error mapping; toast notifications

**Error codes**:
- **400**: Invalid request → Toast: "Invalid move syntax. Try again."
- **401/403**: Not authorized → Toast: "Session expired. Return to lobby?" (action button)
- **404**: Not found → Toast: "Game not found. Returning to lobby." (auto-dismiss)
- **409**: Conflict (game already full) → Toast: "Game already started."
- **422**: Illegal move → Toast: "Illegal move. Check the rules."
- **500**: Server error → Toast: "Server error. Retry?" (action button)
- **Connection timeout**: Toast: "Server unreachable. Retry?" (retry button)

**Implementation**: Error handler in adapter layer; returns human-readable error that UI renders as toast.

---

## Summary of Decisions

| Item | Decision | Status |
|------|----------|--------|
| Language & framework | Python 3.11 + Pygame 2.x | ✅ Resolved |
| Primary dependencies | pygame, requests, python-dotenv, pytest | ✅ Resolved |
| Storage | None (stateless) | ✅ Resolved |
| Testing | pytest + mocking | ✅ Resolved |
| Target platform | Windows, macOS, Linux (cross-platform) | ✅ Resolved |
| Project type | Desktop GUI app | ✅ Resolved |
| Performance goals | < 500ms move latency, < 2% idle CPU | ✅ Resolved |
| Constraints | 1024×768 min, custom sprites, 5 min timeout | ✅ Resolved |
| Scale | 6–8k LOC, 5 screens, 50–100 sprites | ✅ Resolved |
| Asset pipeline | Aseprite/Piskel → PNG → embedded | ✅ Resolved |
| Polling | Thread-based, 1–2 sec interval | ✅ Resolved |
| Error handling | Explicit mapping → toast notifications | ✅ Resolved |

---

## Next Steps

1. **Proceed to Phase 1 (Design)**:
   - Generate data-model.md: state machine definition, screen flow diagrams
   - Generate contracts/server-api.md: HTTP contract expectations
   - Generate quickstart.md: build + run instructions for Python + Pygame

2. **Approval**: Review research.md findings with team; confirm Python + Pygame is acceptable.

3. **Generate tasks**: After design is complete, run `/speckit.tasks` to generate implementation task list.
