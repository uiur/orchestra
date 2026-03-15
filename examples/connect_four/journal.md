# Connect Four -- Build Journal

## Build Process

- **Goal**: Build a Connect Four browser game with AI, sound effects, and adjustable difficulty
- **Approach**: Super-manager split into 3 parallel tracks with managers: engine (game logic + AI), frontend (UI + wiring), audio (Web Audio API sounds)
- Each manager spawned 2 workers (6 total workers running in parallel)
- Workers used isolated git worktrees to avoid conflicts

### Timeline

- Super-manager created feature branch, wrote task specs with interface contracts, spawned 3 managers
- mgr-engine spawned wkr-gamelogic (board state, move validation, win detection) and wkr-ai (minimax with alpha-beta pruning)
- mgr-frontend spawned wkr-board (CSS grid board renderer with drop animations) and wkr-app (application shell with mode switching, scoreboard)
- mgr-audio spawned wkr-sounds (Web Audio API synthesized effects)
- All 6 workers completed and committed within ~5 minutes

### Integration Challenge

- Managers became non-responsive after spawning workers (Claude Code sessions stopped at interactive prompts after their first turn)
- Workers used different module systems: engine/AI used IIFEs with window globals, frontend used ES module imports/exports, audio used ES module exports
- App logic was built against a functional API (createGame, makeMove as standalone functions) but engine was a class (GameEngine with methods)
- Super-manager took over integration: read all 8 files from worker branches, used an integration agent to combine into single self-contained index.html

## What Went Well

- **Parallel execution**: 6 workers produced quality code simultaneously in ~5 minutes
- **Clean decomposition**: engine/frontend/audio split meant almost zero file conflicts
- **Worker output quality**: proper minimax with alpha-beta pruning, polished CSS with animations, synthesized audio
- **Final product** works as a single file with no dependencies

## What Could Improve

1. **Manager lifecycle**: Managers using Claude Code's interactive mode completed one turn and stopped. Need a wrapper script that feeds inbox messages as follow-up prompts, or use a non-interactive mode.
2. **Interface enforcement**: Despite specifying API contracts in task descriptions, workers diverged (ES modules vs IIFE, class vs functions). A shared CONTRACT.md file that workers read before coding would help.
3. **Health monitoring**: Super-manager should actively poll manager pane state every 60s rather than passively waiting for inbox messages.
4. **Consider codex for managers**: Codex runs single prompts to completion without interactive prompt issues. Managers don't need interactivity.

## Outcome

1739-line single HTML file with: game engine, minimax AI (5 difficulty levels), Web Audio API sounds, CSS animations, responsive design. Zero external dependencies -- just open in a browser.
