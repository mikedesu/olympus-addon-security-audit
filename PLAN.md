# Audit Plan

Target: Olympus 1.1.5, commit `5825ed4`. Findings for each stage go in the `session-NNN.md`
of the session that did the work.

- [x] 0. Write `README.md` and this plan.
- [x] 1. Inventory and provenance: pin the commit, confirm what ships (the `Olympus/` folder), check that the TOC loads only files present in the repository, review git history, authors and tags, and look for hidden files, odd binaries and obfuscated or minified code.
- [x] 2. Runtime and sandbox: identify the client and its Lua interpreter, list what the sandbox forbids (files, network, processes) and what an addon can still do, and write the threat model.
- [x] 3. Dangerous-primitive sweep over all shipped Lua: dynamic code loading (`loadstring`, `load`, `RunScript`, `setfenv`, `string.dump`), global environment tampering, hooks and replacements of Blizzard functions, and dynamically built names that could hide calls.
- [x] 4. In-game action sweep: every call that acts as the player (chat, whispers, addon messages, mail, trade, gold, items, group and guild management, chat channels, CVars, macros, bindings, logout), each classified as user-initiated or automatic.
- [x] 5. Inbound message handling: how `Comm`, `Codec`, `Channels` and the core parse what other players send, with validation, size and rate limits, and whether remote input can reach code execution or harmful actions.
- [x] 6. Remote authority model: what the King, Steward, Hands, High Council and the author's signing key can make my client do or show (decrees, net-off, block terms, key rotation), plus a sanity check of the Ed25519 and signature code.
- [x] 7. Privacy: what my client collects and broadcasts (census, location, layers, alts, crafters, treasury, guild bank, inspections, loot notes, bug reports, the bridge for other addons), defaults versus opt-in, and the README privacy table checked against the code.
- [x] 8. User interface and chat integration: chat filters and hooks, chat frames, nameplates, unit frames, the guild frame, right-click menus, popups and gamepad UI, with attention to suppressing non-Olympus chat, clickable links shown to me, and taint.
- [x] 9. Olympus Link and the web side: what the QR code and proof contain, the GitHub Pages site, the Cloudflare worker and its database schema, third-party scripts, and what leaves the game and where it is stored.
- [x] 10. Bundled libraries: LibStub, CallbackHandler-1.0, HereBeDragons and QREncode compared against their upstream sources.
- [x] 11. Developer files that do not ship: `scripts/`, `tests/`, `.github/workflows/` and the web tooling, and what they would do if run.
- [x] 12. Distribution check: the published release zip compared with the audited tag.
- [x] 13. Verdict and recommendations: overall rating, which settings to choose, safe install steps, and residual risks.
