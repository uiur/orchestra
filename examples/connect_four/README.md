# Connect Four

A fully-featured Connect Four browser game with player-vs-player and player-vs-AI modes, built as a demo by orchestra's multi-agent system.

## How to Play

Open `index.html` in any modern web browser. No server, build step, or dependencies required.

## Features

- **PvP mode** -- two players take turns on the same screen
- **PvAI mode** with adjustable difficulty (levels 1-5) powered by minimax with alpha-beta pruning
- **Sound effects** synthesized in real-time using the Web Audio API
- **Drop animations** for pieces falling into the board
- **Win highlight** showing the four connected pieces
- **Responsive design** that works on desktop and mobile
- **Scoreboard** tracking wins across rounds

## Tech

Single self-contained HTML file (~1739 lines). No external dependencies, no framework, no server required. All game logic, AI, rendering, and audio are inlined. Just open it in a browser and play.

## About

This game was built by orchestra's multi-agent system as a demonstration of parallel task decomposition. A super-manager split the work into engine, frontend, and audio tracks, each managed independently with parallel workers, then integrated the results into a single file.
