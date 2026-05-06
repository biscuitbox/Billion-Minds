# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

Pre-implementation. The repository currently contains only `README.md`. There is **no source code, package manifest, build pipeline, or test suite yet** — do not invent or run build/lint/test commands. When the user asks for "the build" or "the tests," confirm what they want set up rather than guessing a toolchain.

Active branch is `4k-mode` (the resolution decision was made on this branch). `main` holds the initial commit only.

## Project Concept

Billion Minds is a planned web experiment: a 4K canvas is repainted with random colors every hour, and each new frame's pixel-level similarity to the **first frame ever generated** is plotted as a percentage line graph over time. The philosophical question — *"can humanity's minds converge?"* — is the framing; the math is a probability visualization.

Read `README.md` for the full rationale. The points below are the hard constraints that any implementation must respect.

## Hard Constraints (from README)

These are load-bearing decisions. Do not change them without explicit user confirmation.

- **Canvas size: 3,840 × 2,160 = 8,294,400 pixels.** This is the upper bound, chosen because 1 billion pixels was rejected as infeasible (browser canvas limits ~16,384px, ~2.79 GB/frame, ~24.5 TB/year). The project name's "Billion" is a philosophical metaphor for ~8 billion humans, *not* a pixel count.
- **Color depth: 24-bit RGB** (~16.77M colors). Probability math throughout the README assumes this.
- **Refresh cadence: every hour, on the hour.** Frames must be reproducible — generation is **seed-based PRNG**, not unseeded randomness, so historical frames can be regenerated rather than stored as raw images.
- **Genesis Frame is immutable.** It is the sole comparison baseline for every subsequent frame; do not introduce rolling baselines or "compare to previous hour" logic unless the user asks.
- **Similarity is exact-match pixel equality** by default. The README mentions Euclidean RGB / ΔE as a *future option* — keep that distinction; don't silently switch the default.

## Expected Numerical Behavior

The expected number of matching pixels between two random 4K frames is `8,294,400 / 2^24 ≈ 0.494`. Most refreshes will produce **0 or 1 matching pixels**. If a similarity calculation yields tens or hundreds of matches, that signals a bug (likely a seeding collision, off-by-one in comparison, or the Genesis Frame being accidentally reused as the current frame) — not a meaningful "convergence" event.

## Planned Stack (Not Yet Chosen)

The README lists candidates: FastAPI · NumPy/Pillow · PostgreSQL + S3-compatible object storage · APScheduler/cron · React + WebGL + D3.js · Docker. Treat these as defaults to propose if the user starts implementation, but confirm before scaffolding — the README explicitly marks them as tentative.

## Environment Notes

- Platform: Windows 11, PowerShell. Use PowerShell syntax for shell commands (`$env:VAR`, backtick line continuation, no `&&` chaining in PS 5.1).
- Project path: `C:\Users\SM\pythonProjects\Billion-Minds`.
- The README is written in Korean; the user communicates primarily in Korean. Match their language unless they switch.
