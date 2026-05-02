<!-- SPECKIT START -->
## Implementation Plan: Desktop Chess Client

**Active Feature**: `001-desktop-chess-client` (v1.0.0)  
**Spec Status**: Complete (clarified)  
**Plan Status**: Phase 1 Design complete (research.md, data-model.md, contracts/, quickstart.md generated)

**Key Context**:
- Language: Python 3.11+ with Pygame 2.x
- Architecture: Clean layers (UI → State → Adapter → Models)
- Screen flow: username entry → lobby (polling) → game board (chess play) → game over
- Backend: ajedrez4-server HTTP REST API (polling-based, 1–2 second intervals)
- State management: Explicit state machine with UI event handling
- Concurrency: Thread-based polling for game state updates
- Dependencies: pygame, requests, python-dotenv; zero external chess logic (delegated to server)
- Testing: Unit (state machine), contract (HTTP mocking), integration (full flows)
- Visual style: Pixel-art sprites, Kathy Rain aesthetic, custom assets

**Design Artifacts**:
- [specs/001-desktop-chess-client/plan.md](specs/001-desktop-chess-client/plan.md) - Implementation plan with tech stack, structure, phases
- [specs/001-desktop-chess-client/research.md](specs/001-desktop-chess-client/research.md) - Phase 0: language/framework decision (Python + Pygame rationale)
- [specs/001-desktop-chess-client/data-model.md](specs/001-desktop-chess-client/data-model.md) - Phase 1: state machine, domain types, polling logic
- [specs/001-desktop-chess-client/contracts/server-api.md](specs/001-desktop-chess-client/contracts/server-api.md) - HTTP API contract (endpoints, polling, error handling)
- [specs/001-desktop-chess-client/quickstart.md](specs/001-desktop-chess-client/quickstart.md) - Setup, build, run, dev workflow

**Next Step**: Run `/speckit.tasks` to generate phase 2 implementation tasks
<!-- SPECKIT END -->
