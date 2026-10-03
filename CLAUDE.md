# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, no-build web app (`index.html`: inline CSS, HTML, and one IIFE `<script>`) that tracks anti-VEGF injection intervals for the right eye (RE) and left eye (LE), projects the next injection dates, and suggests review dates. There is no package manager, bundler, linter, or test suite. To run it, open `index.html` in a browser (or serve the folder statically). To verify a change, load the page and exercise the affected flow. Deployment is whatever serves the repo's `index.html` (it's hosted from GitHub).

## Architecture (all in `index.html`)

- **State** is one in-memory object: `state = {eye, RE, LE, reviews}`. Each eye holds `events` (injections), `interval` (planned weeks, or null to follow the last interval), `drug`, and `shift` (per-slot manual day offsets). `analyse(eye)` derives everything else: it merges injections and reviews, computes the last interval, and projects future dates. The first upcoming injection always uses the last interval (it's usually already booked), and the planned interval applies from the second onwards. Each shift carries on to later dates.
- **Rendering** is imperative: each `render*()` function rebuilds its section's DOM, and `render()` calls them all. After mutating state, call `render()`. `keepInPlace()` preserves scroll position around re-renders.
- **Closures** (clinic closed days, public holidays) are the only thing persisted, in `localStorage` (`clinicClosures`, `ph2027Added`). `DEFAULT_CLOSURES` seeds them, with small one-off migrations applied on load. The intro dialog says these dates are unverified placeholders. Keep that warning accurate if you change them.
- **Patient data is deliberately never stored.** Instead, the "Patient code" feature serialises the essentials into a few words that the user pastes into clinical notes and loads next time. This is why the encoding is delicate (see below).

## Patient code: compatibility rules (read before touching it)

Codes already written into real notes must keep decoding, so:

- `CODE_WORDLIST` is **frozen**. Never add, remove, or reorder a word.
- The code is a bit-packed integer rendered as base-N words (N = wordlist length), with an 8-bit CRC. Layouts are versioned: legacy, v1 (`encodeSnapshot`/`decodeSnapshot`, absolute dates from `EPOCH24`), v2/v3 (`encodeV2`/`decodeV2`, anchor date stored modulo `ROLL`=2048 days, so a code reads correctly for about 5 years). `parseCode` tries the decoders. **A new layout needs a new version; never change an existing encoder or decoder.**
- Current output comes from `buildCodeV2`, which round-trips through `decodeV2` and returns null (the UI then says it can't be encoded) if the result doesn't match. Drugs and OCT results are zeroed out of v3 codes on purpose.
- Applying a decoded code goes through `applySnapshot`, `applySnapshotV2`, or `applyLegacy`, then `finishLoad`.

## Conventions

- Dates are integer day numbers (days since 1970, UTC) via `dayNum`/`toIso`/`todayNum`. Format them with the `fmt*` helpers (en-AU locale). Don't use local-time `Date` arithmetic.
- Escape any user-derived text inserted into HTML with `esc()`.
- Wrap every `localStorage` access in try/catch, as existing code does.
- The app runs on phones and in clinics, so keep it dependency-free and working offline from the single file.
