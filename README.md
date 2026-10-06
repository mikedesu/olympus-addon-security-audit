# Olympus Addon Security Audit

A personal security review of the **Olympus** World of Warcraft addon ("Olympus Guild" on
CurseForge) before installing it. Olympus is the guild addon of Asmongold's Olympus guilds on
*World of Warcraft: Forever*, Blizzard's Classic-style game.

The question this audit answers:

> Does installing Olympus put my machine, my WoW account or my privacy at risk, and does it do
> anything abusive or undisclosed?

## Target

| Item | Value |
|---|---|
| Upstream repository | https://github.com/dnl-gentile/olympus-addon |
| Local clone | `/home/darkmage/src/olympus-addon/` |
| Version audited | 1.1.5 |
| Commit audited | `5825ed41843cd9b303276975a92b752cd48daa69` |
| What gets installed | the `Olympus/` folder only (what `scripts/package.sh` zips) |
| Game clients declared | WoW: Forever 1.60.1 (Interface 16001), Classic Era 1.15 (11509), Anniversary 2.5 (20506) |

To look at exactly the audited code:

```sh
git -C /home/darkmage/src/olympus-addon checkout 5825ed41843cd9b303276975a92b752cd48daa69
```

## Scope

In scope:

- Every Lua file the game loads, as listed in `Olympus/Olympus.toc`, bundled libraries included.
- What the addon sends to other players, and what it accepts from them.
- What other players, guild leaders and the addon's author can make my client do or show.
- The off-game pieces a player can touch: the Olympus Link web page and its backend worker.
- Developer scripts in the repository, in case I ever run them.
- Whether the published release zip matches the audited source.

Out of scope:

- The CurseForge app (Overwolf), WowUp and the WoW client itself.
- Blizzard Terms of Service rulings, beyond noting obvious automation.
- Server infrastructure that is not in the repository. Only the worker's source is reviewed.

## Method

- **Read-only.** No repository code is run on this machine until it is reviewed. The test suite
  runs under LuaJIT, which has full file and OS access, unlike the game's sandbox.
- **Sweeps first.** Pattern searches across all shipped Lua for dangerous primitives: dynamic
  code loading, environment tampering, hooks, and every API call that acts as the player.
- **Then a full read.** Every module is read, grouped by risk area: messaging, remote authority,
  privacy, economy, user interface, Olympus Link.
- **Claims are checked.** The addon's own `README.md` privacy table and `SECURITY.md` are
  compared against what the code does.
- **Evidence.** Every finding cites `file:line` at the audited commit.

## Severity scale

| Level | Meaning |
|---|---|
| Critical | Harms the machine or the account, or hides malicious behavior. |
| High | Lets another player or the author do something harmful through my client, or leaks sensitive data without consent. |
| Medium | Sends personal data by default, or behaves in a way a user would want to know before installing. |
| Low | Minor robustness or privacy issue with limited impact. |
| Info | Neutral observation, documented behavior, or a verification note. |

## Files in this folder

- `README.md` describes the audit.
- `PLAN.md` lists the audit stages as a checklist.
- `session-NNN.md` logs each working session: what was checked, how, and what was found.

## Status

Session 001 (2026-10-06) completed every stage of `PLAN.md`.

**Verdict: no malicious code was found, and the overall risk is low.** The addon cannot touch
the machine, never acts on the account without a click, and keeps personal sharing off until
you say yes.

| Severity | Count |
|---|---|
| Critical | 0 |
| High | 0 |
| Medium | 2 |
| Low | 30 |
| Info | 58 |

The two Medium findings are both about the Olympus channel. The census names guild leaders,
officers and top-level members to everyone on it, by design and without an opt-out. And while
the channel is unsealed, a few colluding strangers can push raid-warning text to everyone.

Recommended settings, in-game checks and a recipe for re-checking future updates are in
`session-001.md`, Stage 13. The verified release zip has this SHA-256:

```
386dc19579803f220d65a76b4d3728af9d3644009ef6193341f0d370152a710f  Olympus-1.1.5.zip
```
