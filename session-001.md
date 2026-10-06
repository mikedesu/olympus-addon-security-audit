# Session 001

- Date: 2026-10-06
- Auditor: Claude (Opus 5.5), in Claude Code, for darkmage
- Target: Olympus 1.1.5, commit `5825ed41843cd9b303276975a92b752cd48daa69`, clone at `/home/darkmage/src/olympus-addon/`
- Rule for this session: read-only. No repository code was executed on this machine.

## Stage 0: README and plan

Wrote `README.md` (scope, method, severity scale) and `PLAN.md` (stages 0 to 13).
The audit files live in this folder, not in the addon clone, because the clone has its own
`README.md` and should stay pristine. This folder is not a git repository yet.

## Stage 1: Inventory and provenance

What I checked: git state, tags, authors and signing; the TOC load list against the files on
disk; hidden files, binaries and XML; and obfuscation markers (escaped byte strings, hex
escapes, long lines, encoded blobs, non-ASCII and invisible Unicode).

| ID | Severity | Finding |
|---|---|---|
| S1-1 | Info | `HEAD` is `5825ed4`, the target of the annotated tag `v1.1.5`. The working tree is clean. |
| S1-2 | Info | 169 commits between 2026-09-23 and 2026-10-05. Authors: Daniel Gentile (165), Jesse Drummond (3), KonigTX (1). 33 merges were made in GitHub's web UI. No commit is GPG-signed, which is normal for hobby projects but means authorship rests on GitHub accounts only. |
| S1-3 | Info | The project is 13 days old, about 58,700 lines of Lua, and moves fast (several releases a day at times). A remote branch named `claude/...` suggests AI-assisted development. The realistic risk from this is bugs rather than malice, and each update can change behavior. |
| S1-4 | Info | What ships is the `Olympus/` folder (`scripts/package.sh`). All 66 files in `Olympus.toc` exist, no `.lua` file sits outside the TOC, there are no `.xml` files, and no hidden files. The six `.tga` files are real Targa images (`file` output), plus one license text. |
| S1-5 | Info | No obfuscation found. No `\xHH` escapes. Only two decimal-escaped strings: a zero-width-space constant (`Codec.lua:509`) and the standard ASN.1 prefix for SHA-256 signatures (`Sign.lua:145`). Lines over 300 characters are locale strings, two help texts, one list of module names (`Core.lua:1800`) and an RSA public key (`Sign.lua:123-124`). No bidirectional-override or invisible Unicode in any code. Non-ASCII bytes are middle dots and translated words. |
| S1-6 | Info | Hardcoded keys are public keys only: an RSA modulus that verifies the author's signed lists (`Sign.lua:123`) and an Ed25519 certificate-authority key (`Link.lua:81`). The Discord bot's key is still the placeholder `PASTE-THE-BOT-PUBLIC-KEY-HEX-HERE` (`Link.lua:75`), and the code says every Link code is refused until it is set, so Olympus Link looks dormant in this release (checked in stage 9). |

Stage 1 result: **clean.** Nothing hidden, packed or obfuscated. The release tag matches the audited commit locally.

## Stage 2: Runtime, sandbox and threat model

### Which client and which Lua

- **WoW: Forever is Blizzard's own game.** The addon's README says it targets "World of Warcraft:
  Forever" and its beta 1.60.1, which matches `## Interface: 16001` in the TOC. The client model
  in `tests/fixtures/forever-api.lua` was extracted from the client's own UI source (build 70205).
- **It runs on the modern retail engine.** `scripts/forever-api.lua` (header) reads the client
  as the Mainline family with game type "Camelot" (`WOW_PROJECT_CAMELOT`). Public sources agree
  that Forever uses the modern retail addon API and its combat restrictions.
- **The interpreter is Blizzard's embedded, modified Lua 5.1.** Every modern WoW client
  (retail, Classic Era, Anniversary and now Forever) embeds the same sandboxed Lua 5.1. The code
  fits this: no Lua 5.2+ syntax (`goto`, labels, `//`, bitwise operators), and it relies on
  5.1/WoW idioms such as global `unpack`, the `bit` library, `wipe`, `strsplit` and `C_Timer`.
  The developers test offline with LuaJIT, which is also Lua 5.1 compatible.

You can confirm the interpreter and the sandbox in your own client with these chat commands:

```
/run print(_VERSION)
/run print(io, require, dofile, loadfile, os and os.execute)
```

The first should print `Lua 5.1`. The second should print `nil` five times.

Sources:
[GitHub topic world-of-warcraft-forever](https://github.com/topics/world-of-warcraft-forever),
[Blizzard forums: WoW Forever and addon restrictions](https://us.forums.blizzard.com/en/wow/t/wow-forever-and-addon-restrictions/2371019),
[Blizzard forums: Anyone test addons?](https://us.forums.blizzard.com/en/wow/t/anyone-test-addons/2352856),
[wowguide.net: Do addons work in WoW Forever?](https://wowguide.net/en/guides/wow-forever-addons).
Several of these are fan sites or player posts, not Blizzard, so treat release details as unconfirmed.

### What the sandbox forbids

- No file access. There is no `io` library, no `dofile`, `loadfile`, `require` or native-library
  loading. The only file an addon affects is its own SavedVariables file, which the **client**
  writes on logout or `/reload`. Olympus declares one: `OlympusDB` (TOC line 6), stored
  account-wide in `WTF/Account/<account>/SavedVariables/Olympus.lua`.
- No network access. No sockets, HTTP or URL opening. An addon can only show text or a QR code
  that you choose to act on outside the game.
- No processes. No `os.execute`, environment variables or program launching.
- No access to your password, Battle.net session or anything else on the computer.

### What an addon can still do (the real threat model)

1. **Act as you in the game.** Send chat, whispers and hidden addon messages, join chat
   channels, invite to or accept groups, use guild powers your rank has, fill in the mail form,
   move or delete items. Blizzard requires a real key press or click for some of these (for
   example `/who` searches and some public chat), and blocks most combat actions entirely.
2. **Run other people's code inside the sandbox** if it feeds received text to `loadstring`.
   That would hand everything in point 1 to whoever sends the text.
3. **Leak what the client knows about you** to other players: location, layer, gear, gold,
   professions, guild data, and other characters' names from the account-wide saved data.
4. **Mislead you with its interface.** Fake system or GM text, fake links, URLs to copy, or
   pre-filled mail.
5. **Slow or break the game.** Heavy CPU work, memory growth, or taint that makes Blizzard block
   the protected actions of its own interface.
6. **Get the account in trouble.** Automation or spam that breaks the Terms of Service.
   Messages sent through the "logged" addon-message API are recorded by Blizzard.

### Paths that reach outside the game

The addon itself has none. The only indirect paths are:

- the install channel (the CurseForge app or a downloaded zip), covered in stage 12;
- developer scripts in the repository, which only matter if you run them (stage 11);
- a URL or QR code you choose to open in a browser (Olympus Link, stage 9);
- data at rest in the SavedVariables file, which is plain text on disk.

Stage 2 result: **the machine-level risk from the addon code is effectively nil by construction.**
The audit therefore concentrates on threat-model points 1 to 6 inside the game.

## Stage 3: Dangerous-primitive sweep

What I checked, across the 66 shipped Lua files: code loading, environment tampering, global
writes, dynamic global lookups, hooks and replacements of Blizzard functions, chat filters and
menu changes.

| ID | Severity | Finding |
|---|---|---|
| S3-1 | Info | **No code is ever loaded from text.** Zero uses of `loadstring`, `load`, `RunScript`, `setfenv`, `getfenv`, `dofile`, `loadfile` or `string.dump` in shipped code. The only hit is a comment (`Channels.lua:304`). The classic addon attack, where other players send code your client runs, is not possible. |
| S3-2 | Info | **No tampering with the global environment.** No metatable or `rawset` on `_G`. Every `_G[...]` lookup built at runtime takes its name from a fixed list in the code: Blizzard window names (`Who.lua:139`), macro-icon functions (`Workshop.lua:2526`), mail-subject strings (`Treasury.lua:793`), chat-tab labels (`Channels.lua:249`) and frame names. None takes a name from a message. |
| S3-3 | Info | **The only globals created are Olympus's own:** `OlympusDB` (`Core.lua:1563`), `OlympusBridge` (`Bridge.lua:8`, documented), the slash commands `/olympus`, `/oly`, `/ol`, `/olc`, `/oll`, popups named `OLYMPUS_*`, and named Olympus frames. Seven top-level assignments that looked global are forward-declared locals. No Blizzard function is replaced. |
| S3-4 | Info | **Eight post-hooks (`hooksecurefunc`)**, which run after Blizzard's function and cannot change or block it: `C_FriendList.SendWho` (`Who.lua:193`), `CompactUnitFrame_UpdateName` (`Nameplates.lua:343`), unit-frame updates (`Borders.lua:553`, `563`), and `SendMail`, `TakeInboxMoney`, `TakeInboxItem`, `AutoLootMailItem` (`Treasury.lua:2694-2698`). The mail hooks mean the addon **observes** the mail you send and take. Stage 7 checks what it records. |
| S3-5 | Info | **No chat message filters.** `ChatFrame_AddMessageEventFilter` is never used, so the addon has no standard way to hide your normal chat. Right-click entries use Blizzard's supported `Menu.ModifyMenu` (`PlayerMenu.lua:184`). `HookScript` calls only add behavior: docking beside the guild, friends and communities windows, and the world map. |
| S3-6 | Info | An optional "Hide" button for Blizzard's beta Issue Reporter (`UI.lua:366-410`). It is off until you choose it, and it never touches the reporter in combat or with the gamepad UI. |

Stage 3 result: **clean.** No path exists for received text to become running code.

## Stage 4: In-game action sweep

What I checked: a search of all shipped Lua for about 150 API names that act as the player,
plus a list of every namespaced Blizzard function the addon touches. I then read the code
around each hit to see what triggers it.

### Never called anywhere

The addon calls none of the following. Sending mail and taking mail money or items
(`SendMail`, `TakeInboxMoney`, `TakeInboxItem`, `AutoLootMailItem`) are only post-hooked, to
observe them (S3-4). Also never called: cash on delivery, returning or deleting mail, starting
or accepting trades, picking up, using, splitting, deleting, buying or selling items, guild-bank gold or items, master loot and loot rolls, the auction house, casting or
targeting, CVars, macros, saved key bindings, logout or quit, reporting players to Blizzard,
guild notes, the message of the day, rank changes, friend and ignore list edits, and
Battle.net whispers. `C_Calendar.AddEvent` appears only in a comment saying the addon cannot
write to the calendar (`Week.lua:14`).

### Actions it does take

| ID | Severity | Action | Trigger and conditions |
|---|---|---|---|
| S4-1 | Info | **Whispers as you** (`SendChatMessage`, always `"WHISPER"`, 6 sites) | All on your click: a Join Olympus request whose text sits in an edit box you can change, with a cooldown and a do-not-contact list (`Recruit.lua:339`); an officer's decline or pointer reply (`Recruit.lua:570`, `585`); the whisper popup (`UI.lua:1706`); a Lord's "Send both" mentor pair (`Members.lua:213-214`). It never sends say, yell, guild, raid or channel chat. SECURITY.md says it "never sends chat for you unless you type it". That is nearly right: a few canned whispers go out on your click without you typing them. |
| S4-2 | Info | **Guild invite and removal** | Invite (`Recruit.lua:547`) and removal (`Members.lua:80-96`, `Dues.lua:1340-1343`) run only on an officer's click, with a confirmation popup, and only if the game grants that rank the power. Removals are rate-limited. |
| S4-3 | Low | **Fills in gold for dues** | Only when you click "Send this week's dues": it fills the open mail form (recipient, subject, gold, "Send Money" rather than cash on delivery) or your gold in an open trade (`Dues.lua:1612-1647`). It never presses Send or Trade. The recipient is hardcoded to the Treasurer's characters (`Core.lua:833-848`), and a trade is filled only if the partner is one of them. **The amount comes from the King or his Steward over the network**, capped at 1,000 gold (`Dues.lua:77`, `245`). Read the amount the addon prints before you press Send. |
| S4-4 | Low | **Automatic group actions for the layer hop** | Only while you have a hop request open: your client accepts an invite from a helper it asked whose rank it vouches for, and hides the game's invite popup (`Hop.lua:503-528`). It leaves the group after the hop (`Hop.lua:530-537`). As a helper with layer help on, which is opt-in, it whispers offers and invites the guest (`Hop.lua:189-236`). Invites from anyone else get the game's normal popup. SECURITY.md admits a forged census rank can get an invite auto-accepted for someone who asked to hop. |
| S4-5 | Info | **Hidden chat channel** | Joins one channel, with a password once sealed (`Comm.lua:817`). It removes the channel from your chat windows (`Comm.lua:469`) and moves it after General and Trade so they keep their numbers (`Comm.lua:1182-1207`). If the game makes your client the channel owner, it uses owner powers only to undo a lock: it restores the password, turns moderation off, and unbans and unmutes (`Comm.lua:609-628`). It never bans, mutes, kicks or locks anyone. SECURITY.md's "never uses that power" is loose wording for this. |
| S4-6 | Info | **`/who` searches** | Through `C_FriendList.SendWho` (`Who.lua:103`), which Blizzard limits to a real click. While Olympus searches, results go to the addon instead of the Who window, and the setting is put back afterwards (`Who.lua:97-162`). |
| S4-7 | Info | **Inspecting nearby players** | `NotifyInspect` (`Inspect.lua:223`) for the Royal Inspection and officer patrols. Opt-in according to the README. Stage 7 confirms the default. |
| S4-8 | Info | **Hidden addon messages** | All sends go through `Comm.lua:323`, plus one at `Treasury.lua:2810`. Free text (decrees, Board notes, chat lines, net-off reasons) goes through Blizzard's *logged* addon-message API, so Blizzard keeps a record of it. |
| S4-9 | Info | **Other** | Temporary key overrides while you type in the Olympus chat box, never saved (`ChatWindow.lua:1509`). `ReloadUI` only from a button on its gamepad notice (`Gamepad.lua:145`). `LoadAddOn` only to load Blizzard's own world map (`Bootstrap.lua:91`). |

Stage 4 result: **no harmful or hidden actions.** Every action that touches gold, the guild or
chat needs your click. Two behaviors deserve awareness. The dues helper fills in an amount
someone else chose (S4-3), and the layer hop auto-accepts a vetted invite while you are hopping (S4-4).

## Stage 5: Inbound message handling (the messaging core)

Files read in full by a delegated reviewer: `Bootstrap.lua`, `Core.lua`, `Comm.lua`,
`Codec.lua`, `Channels.lua`, `Versions.lua`, `Diagnostics.lua`, `Data.lua`, `Roster.lua`. I
re-read the census builder (`Roster.lua:60-140`) and the channel-owner code myself.

| ID | Severity | Finding |
|---|---|---|
| S5-1 | Medium | **The census broadcasts guildmates' details by default.** One elected member per guild sends the leader's and every officer's name, online state, days offline, class and level, the five highest-level members (name, level, class), rank names and counts, and online counts per zone (`Roster.lua:60-136`, verified). It goes to everyone on the Olympus channel, which anyone can join while the channel is unsealed (`Comm.lua:462-463`). Nobody named is asked, and there is no opt-out except leaving the guild. The README documents it (lines 1551-1553). If you are an officer or one of the top five levels, your details go out even if *you* never install the addon. |
| S5-2 | Low | **Channel-owner powers are used automatically to undo locks** (`Comm.lua:555-628`). If the game makes you the channel owner, your client clears the password (or resets it to the key), turns moderation off, unbans and unmutes up to 20 players, and always unbans the King and the Treasurer. Each action runs at most once a minute. This contradicts SECURITY.md line 99 and the in-game notice (`Locales.lua:724`), which say Olympus never uses these powers. |
| S5-3 | Low | **A crafted census report could briefly stall clients.** Reports can be up to 6,600 bytes. A long field without separators makes a Lua pattern scan take quadratic time (`Codec.lua:91`, `111`, `283-291`). The estimate is under a second per report, unmeasured, and the per-sender admission limit caps the rate (`Comm.lua:1248-1272`). |
| S5-4 | Low | **Anyone can learn that you run Olympus and which version** by asking your addon (`Versions.lua:261-274`). Rate-limited, and never answered to players you ignore. Documented. |
| S5-5 | Low | **Fake guild reports can grow the saved-variables file.** One new stored guild per 15 minutes per character, kept 7 days, with no cap on the count (`Data.lua:11`, `156-157`, `389`). |
| S5-6 | Low | **Password-lock griefing.** Whoever owns the unsealed public channel can set a password. The addon then retries with backoff, and the game's password prompt may appear: press Cancel. |
| S5-7 | Info | **No remote code execution and no remote-chosen dispatch.** Messages are routed by a two-letter type through a local table filled only with literal types (`Comm.lua:1399`). Remote keys land only in fresh plain tables. Nothing remote writes into the addon's namespace, its settings or `_G`. |
| S5-8 | Info | **Inbound paths.** One addon prefix, `OLYMPUS` (`Comm.lua:1291`). Handlers check the distribution (channel, guild, whisper) they accept. Outside an Olympus guild only two messages are processed: Join-screen answers and the author's signed titles list. |
| S5-9 | Info | **Rate limits everywhere.** Inbound: 60 messages at once then 2 per second per sender, piece and chat caps (`Comm.lua:1248-1272`, `Codec.lua:584-601`, `Channels.lua:16-20`). Outbound: one message every 1.2 seconds, hello every minute, census every 170 seconds. |
| S5-10 | Info | **Channel key handling.** Any officer of your guild can switch everyone's key over guild chat, and officers' clients hand the key to any guildmate who asks (`Comm.lua:1352-1381`). The key sits in plain text in your saved variables (`ns.rdb.realmKey`). Every member of every sealed guild holds it anyway. |
| S5-11 | Info | **No remote URLs.** Update prompts show fixed text. The only URL is a hardcoded CurseForge link inside whispers you send yourself (`Versions.lua:355`, `487`). |
| S5-12 | Info | **Bug reports** contain your name, guild, realm, server ID, client, known peers, the channel owner, every chat channel you are in, errors, and the last 25 log lines, which can name other addons. They never contain the channel key or Link keys. They are sent only when you press Send, after a preview. The author can open a "please send a bug report" window on your screen at most once a minute, but sending still needs your click. |
| S5-13 | Info | **Guild notes are never read.** The public and officer note values are discarded (`Roster.lua:72`, verified), and no note API is used anywhere. |

Stage 5 result: **no path from a message to code or to an action as you.** The census sharing
(S5-1) is the main privacy point, and it is documented.

## Stage 6: Remote authority model and cryptography

Files read in full by a delegated reviewer: `Sign.lua`, `Ed25519.lua`, `Keys.lua`, `King.lua`,
`Decree.lua`, `Court.lua`, `Moderation.lua`, `Vox.lua`, `Acts.lua`, `Chronicle.lua`,
`Nominees.lua`, `Filter.lua`. I re-read the decree gate (`Decree.lua:80-255`) myself.

### Who can make your client do what

| Role | How it is checked | What it can make your client show or do |
|---|---|---|
| The King ("Asmongold Asmongler"; Horde "Duskmonkey Boneback") | Server-stamped sender name, hardcoded (`Core.lua:889`) | Decrees and calls as raid warnings and popups, polls, writs, court calls, the untabarded list, his map crown, a key rotation that moves your channel, hiding players, editing shared block terms |
| A Steward (up to 3 per faction) | The author's signed titles list, plus name | Most of the King's tools, the treasury settings and the dues amount |
| A Hand (up to 40 per list) | An unsigned list from the King's or a Steward's client, plus name | Summons, inspections, agenda, polls, decrees, hiding at a lower rank, block terms |
| A High Councillor | The author's signed council list, plus name | Hiding, block terms, a silver chat mark |
| The author ("Faladoriel Skylance") | Name, plus his offline RSA key | By name: roll calls (opt-in), an update popup, a bug-report request window. By key: the signed lists that name the High Council, Stewards and approved guilds |
| Your guild's officers | The server's roster | A new channel key over guild chat, which moves your channel |
| Lords and Captains of other guilds | Census votes, which colluding characters can fake | Decree raid warnings, Captains and Lords chat lines, an auto-accepted hop invite after you ask to hop |
| Anyone on the channel | None | Olympus chat lines and census reports |

None of these roles can run code on your client, move your gold or items, or send visible chat as you.

| ID | Severity | Finding |
|---|---|---|
| S6-1 | Medium | **Strangers can put raid warnings on your screen while the channel is unsealed.** A decree is up to 120 characters, shown as a raid warning with sound (`Decree.lua:88-100`). Census-verified Captains may send them, and SECURITY.md admits one or two colluding characters can fake that rank. Limits are one per sender per minute and six per minute in total for census-ranked senders (`Decree.lua:242-251`, verified). Formatting codes are stripped, so there are no clickable links, but plain-text URLs and scam wording are possible. Blizzard logs the text. Hands can also push agenda titles (60 characters plus a popup) and poll questions (90 characters). Sealing the channel with `/oly key` shuts strangers out. |
| S6-2 | Low | **The Horde King's realm is trust-on-first-use** (`Core.lua:891`, `909`). Until learned, any "Duskmonkey Boneback" counts as King. The Alliance side is unaffected. |
| S6-3 | Low | **One Hand or High Councillor can hide any non-leader character, or any guild but the King's, on every client's Olympus screens for 30 days.** It is checked by name, with no signature (`Moderation.lua:576`). It never touches normal Blizzard chat. The only visible change there is that the hidden player's Olympus mark disappears. |
| S6-4 | Low | **The shared block-term list is on by default**, and anyone at Hand level or above can edit it (`Filter.lua:96`). It hides only Olympus addon text, never say, trade, guild or whispers, and never matches names. Edits are logged with the editor's name. Opt out with `/oly filter shared off`. |
| S6-5 | Low | **The channel key comes from weak randomness**, mostly the clock and `math.random`, cut to 80 bits (`Keys.lua:319-325`). It only protects the channel's name and password, and the channel is not encrypted anyway. |
| S6-6 | Info | **A key rotation only moves your channel.** It is accepted by whisper only from the King or a Steward, and by guild chat only from your own officers. Your client whispers your guild name back to the sender, and an officer's client relays the key to the guild. No settings change and no chat is sent. |
| S6-7 | Info | **The signature code is sound.** The council lists use RSA-2048 with e=3, PKCS#1 v1.5 and SHA-256, and the whole padded block is rebuilt and compared, which defeats the known e=3 forgery (`Sign.lua:151-178`). `Ed25519.lua` is a port of TweetNaCl with the RFC 8032 checks added: S below L, canonical point decoding and small-order rejection (`Ed25519.lua:625-784`). Lists are verified before anything is stored, on every path. |
| S6-8 | Info | **No private key material** in the repository or its history. The test seeds were re-derived and are throwaway, and they differ from the production keys. |
| S6-9 | Info | **Hardcoded names grant only documented roles.** Besides the King, Treasurer and author there are the Treasurer's mail character, two built-in approved guilds (OLYMPIAN and OLYMPIANS) and one built-in excluded guild ("Olympus Defense Force"). Developer flags are never set in shipped code. |
| S6-10 | Info | **The author's key is the root of trust.** Whoever holds it can name Stewards, who can set the dues amount, hide players, rotate keys and send decrees. It still cannot run code or act as you. |

Stage 6 result: **no backdoor and no hidden privilege.** The trust model is unusually well
documented. Its weak point is the census, which lets a few colluding characters borrow
Captain rank to push raid warnings to everyone on an unsealed channel.

## Stage 7 (part A): Location, social and crafting privacy

Files read in full by a delegated reviewer: `Positions.lua`, `Map.lua`, `Layers.lua`,
`Hop.lua`, `Zones.lua`, `Who.lua`, `Inspect.lua`, `Alts.lua`, `Members.lua`, `Consent.lua`,
`Recruit.lua`, `Crafters.lua`, `Workshop.lua`. I re-read the consent storage
(`Consent.lua:315-380`, `Layers.lua:44-58`) and the hop group logic (`Hop.lua:189-292`,
`470-640`) myself.

### What goes out, and when

| Data about you | Who receives it | Default |
|---|---|---|
| Exact map position | Your guild | Off. Only with `/oly share` |
| Zone, layer, rank and guild | The Olympus channel | Off until you say yes (officers and a 1-in-8 sample announce) |
| A layer-hop request: zone and wanted layer | The Olympus channel | Only when you click |
| A layer-hop offer | The asker | Off until you say yes to layer help and location |
| Alt links | The Olympus channel | Only after you link characters on both of them |
| Your account's character list | Nobody (kept locally) | Never sent |
| Crafter listing and recipes | The channel, and the asker | Off until you say yes, per profession |
| Join Olympus answers (open guilds, up to 2 officers each) | Any asker | **On for every member** |
| Your name, level and class if you lead, officer or are top-5 level | The Olympus channel | **On, nobody asked** (S5-1) |
| Royal Inspection report | Whoever called it | Off until you say yes |
| Officer patrol findings | Your guild's officers | Off until you say yes |
| Roll-call answer (version, client) | The author | Off until you say yes |

| ID | Severity | Finding |
|---|---|---|
| S7A-1 | Low | **Your privacy answers apply to the whole account.** Every answer is one key in the account-wide `OlympusDB` (`Consent.lua:318-376`; `Layers.lua:46-49` says "account-wide"; verified). A Yes on one character applies to every character in an Olympus guild on that account. The README does not say so. Only the crafter listing is per character. This matters if you keep alts you would rather not have located. |
| S7A-2 | Low | **"Always invite" for layer help invites anyone who asks**, member or not (`Hop.lua:195-262`, verified: no guild check). It is opt-in. Guests without the addon are not removed automatically. |
| S7A-3 | Low | **A timing edge case can make you leave a friend's group.** During your own hop, if you accept a friend's invite before the game has named the group's members, the addon can take it for the hop group and leave it once your layer changes (`Hop.lua:589-593`, `606-614`, verified logic). This is unconfirmed in game. |
| S7A-4 | Low | **Alt claims carry an account-wide timestamp** that can tie linked characters to one account, even after you unlink one (`Alts.lua:84-90`, `266`, `285-290`). This applies only after a manual link. |
| S7A-5 | Low | **Census zone counts include members who said No.** Only the reporter's own choice gates the counts (`Roster.lua:110-117`, verified). In a small guild a count can point at one person. The README's "nothing relays another player's zone" (line 1650) is inaccurate here. |
| S7A-6 | Info | **The layer is detected locally** from creature GUIDs of your target, mouseover and nameplates (`Layers.lua:29-34`). |
| S7A-7 | Info | **Join Olympus answers are automatic for every member.** Your client tells any asker the King's open "gates", up to 8 guilds with room and up to 2 online officers of each (`Recruit.lua:420-486`). This is missing from the README's list of what goes out without a question. |
| S7A-8 | Info | **`/who` runs only on a click**, with a 10-second cooldown. It briefly diverts results from the Who window. At login it sets the "who to UI" option to off, which can override another addon's choice. |
| S7A-9 | Info | **Inspections.** Patrols are off by default and last one session, at most one inspect every 1.5 seconds. The Royal Inspection is opt-in and runs at most once every 30 minutes. Up to 2,000 inspected players are kept for 14 days. You cannot opt out of *being* inspected by others, and the King can publish untabarded players' names. |
| S7A-10 | Info | **Crafters only reads profession data.** No crafting action exists anywhere. While you are listed, anyone can request your full recipe list. |
| S7A-11 | Info | **A daily local jab at non-members.** Outside Olympus, the addon prints a line once a day such as `<guild>? Disband immediately.` (`Recruit.lua:374-382`, `Locales.lua:220`). It sends nothing. |
| S7A-12 | Info | **Decrees carry the sender's exact map position** (`Decree.lua:29-35`). Documented. |

Part A result: **every optional send checks its consent flag, and nothing about your account's
other characters is sent unless you link them.** Know that a single Yes covers the whole account.

## Stage 7 (part B): Gold, mail, trade, guild bank, loot and backups

Files read in full by a delegated reviewer: `Treasury.lua`, `Dues.lua`, `Bank.lua`,
`Backup.lua`, `Letters.lua`, `Loot.lua`, with README lines 906-1238 and 1536-1667. I re-read
the dues code myself (`Dues.lua:236-330`, `1296-1360`, `1580-1690`) to confirm the findings
marked "verified".

| ID | Severity | Finding |
|---|---|---|
| S7B-1 | Info | **"Never moves items or gold" holds.** The only code that touches gold is the dues fill on your click (S4-3). The four mail post-hooks return at once unless you are a treasury keeper or on the Treasurer's account (`Treasury.lua:755`, `879`, `933`). `GetInboxText` is never called. Verified. |
| S7B-2 | Low | **The dues amount is set remotely and can apply to the current week.** `Dues.TakeAmount` stores the sender's `before` field as this week's amount (`Dues.lua:310-323`). The README's "a new amount starts at the next weekly reset" therefore holds only for an honest sender. The amount is accepted only from the King's pinned name or a Steward, and is capped at 1,000 gold. Stewards are named by the author's signed list, so the author's key sits in this chain. Verified. |
| S7B-3 | Low | **Dues payments feed guild removal.** The Treasurer's client whispers each payer's name, gold this week and hours since the last payment to the King, the Steward, and that guild's Captains and Lord (`Dues.lua:776`). Payers are not asked. Captains and Lords get an "unpaid" filter and a Remove button with a confirmation for members below the amount (`Dues.lua:1305-1343`). Mail the Treasurer's mail character has not opened yet counts as unpaid. This happens through other people's clients whether or not you install Olympus, and the README documents it (lines 1106-1124). |
| S7B-4 | Info | **Treasury keepers' books.** A keeper's addon records trades and mail with the treasury, and shares them only after the keeper says yes (`Treasury.lua:1091-1109`). The King or Steward can make any character a keeper remotely. That character's addon then starts a local book at its current gold, but still asks before sharing (`Treasury.lua:543`, `2341-2372`). If you donate, your name and amount appear in the keeper's shared book. This is documented. |
| S7B-5 | Info | **Guild bank snapshot.** It is recorded locally when an Olympus member opens a guild bank (`Bank.lua:293-309`). It is sent only after a guild master, officer or keeper says yes, by whisper to the King, Steward and Hands (`Bank.lua:484-549`). Consent is written only by the popup or slash commands, never by a message or a backup restore (`Bank.lua:588-591`). |
| S7B-6 | Low | **Backup restore from pasted text** can change local state after a confirmation that lists the changes: blocked names, some toggles, book lines (`Backup.lua:308-450`). It is parsed as data, never run as code, and never sets consent or the channel key. The only risk is being talked into pasting a crafted backup. |
| S7B-7 | Low | **Bank tab names from a keeper are only shortened** (`Bank.lua:188`), so they can carry color or texture codes. This is cosmetic. |
| S7B-8 | Info | `Letters.lua` shows release-note popups built from the addon's own text, and uses no mail. `Loot.lua` calls no loot API. Its notes and points travel by guild addon messages, are accepted only from officers, and have formatting stripped (`Loot.lua:98-101`, `318`, `683`). |
| S7B-9 | Info | **What an ordinary member sends from this area automatically:** a timestamp-only request on the channel when the King shows the donor ranking (at most twice a session, `Treasury.lua:2620-2628`), and the Loot page's request for missing time ranges. Your gold (`GetMoney`) is read only for the local "enough gold" check and is never sent. |
| S7B-10 | Info | `TreasurerCharacter()` matches the name "Pyralis Ashandar" on any realm (`Treasury.lua:501-509`). A namesake on another realm would log incoming mail locally. Nothing is sent. |

Part B result: **no hidden or harmful money behavior.** The dues system is the area most
worth understanding, and it is a guild policy question more than a software risk.

## Stage 8 (part A): Chat integration and social UI

Files read in full by a delegated reviewer: `ChatWindow.lua`, `Bridge.lua`, `PlayerMenu.lua`,
`Dialog.lua`, `Answers.lua`, `AnswerBank.lua`, `Board.lua`, `Week.lua`. I re-read the link
sanitizer (`Codec.lua:410-450`) and the Board whisper (`Board.lua:636-644`) myself.

| ID | Severity | Finding |
|---|---|---|
| S8A-1 | Low | **Links in Olympus chat can carry a fake name or color.** A sender can show "[Thunderfury]" that actually links another item (`Codec.lua:446-447`, verified). Every line keeps its `[Olympus] [Name] <Guild>:` prefix, so a whole fake system or GM line is impossible, and hovering shows the real item. You only see these lines after turning the Olympus chats on. |
| S8A-2 | Low | **A Board card click opens the game's chat box** with "/w Name" (`Board.lua:640-642`, verified). The code's own comments say this can taint the chat box so `/cast` and `/use` lines are blocked until `/reload`. Click only. It contradicts README line 546, "the game's chat box, which Olympus never opens". |
| S8A-3 | Low | **The Olympus Chat tab takes over the Enter key** while it is shown (`ChatWindow.lua:1509`). A line typed then goes to the Olympus channel, which is public if unsealed, rather than to guild, party or a whisper. Only your first line per channel asks for confirmation. Documented. |
| S8A-4 | Info | **`OlympusBridge` hands Olympus chat lines to any installed addon**, including Captains and Lords lines your rank can read (`Bridge.lua:58-66`). It is read-only: nothing can push data into Olympus through it. Disclosed in the README. |
| S8A-5 | Info | **A raised Board flag re-sends itself** until it expires (an hour, or 30 minutes for camps), and answers anyone's Board request. It starts only when you raise a flag. |
| S8A-6 | Info | **No chat suppression or auto-replies.** No chat message filter exists anywhere. Answers is a fixed local list with no URLs that only fills a text box. Every popup does what its text says. Right-click entries act only on a click. |

Part A result: **nothing hides, edits or sends your normal chat.**

## Stage 8 (part B): The main window and Blizzard's frames

Files read in full by a delegated reviewer: `UI.lua`, `Views.lua`, `GuildFrame.lua`,
`Borders.lua`, `Nameplates.lua`, `Gamepad.lua`, `GamepadRegistry.lua`, `ViewAs.lua`. I
re-read the chat-marks code (`Borders.lua:900-1010`) and the error handler
(`Bootstrap.lua:40-93`) myself.

| ID | Severity | Finding |
|---|---|---|
| S8B-1 | Low | **Your normal chat gets marks, on by default.** Since 1.1.5 the addon registers Blizzard's sender-name callback (`ChatFrameUtil.AddSenderNameFilter`, `Borders.lua:1006`). It puts a small image before Olympus members' names in say, yell, whispers, party, raid, General, Trade and guild chat. It can only add an image: `ChatClean` rejects anything but texture and atlas codes, so no links or formatting get through (`Borders.lua:907-914`). It never hides or rewrites a message. Turn it off with `/oly chatmarks off`. Verified. |
| S8B-2 | Low | **Marks are cheap to earn.** Any guild whose name contains "olympus" counts, and one or two self-reports for a made-up guild earn the star or a guild master's bronze (code comments, `SECURITY.md` lines 65-69). The effect is cosmetic, but a stranger in Trade chat could look "verified Olympus" on your screen. Do not treat marks as proof of identity. |
| S8B-3 | Low | **Popup taint.** Olympus adds about 70 entries to Blizzard's shared popup table. The addon's own notes say this taints the shared popup state (`GamepadRegistry.lua:93`). Taint is Blizzard's flag for addon-touched code, and it can occasionally block a protected Blizzard popup. This is common addon practice. |
| S8B-4 | Info | **Touches on Blizzard's interface only add things.** Three post-hooks, `HookScript` only (never `SetScript` on a Blizzard frame), textures on unit frames and nameplates placed only out of combat, and forbidden nameplates skipped. |
| S8B-5 | Info | **No settings changes.** No CVar, key binding, macro, action bar or UI scale is changed, and the addon never logs out or quits. It uses no secure button templates anywhere, so remote data cannot reach a button that casts or uses items. |
| S8B-6 | Info | **Other players' text is cleaned.** All escape codes are stripped before any handler runs (`Comm.lua:1299`). The exception is Olympus chat lines, which keep only whitelisted item, spell, enchant and quest links (`Codec.lua:417-447`). Plain text can still be *worded* to look official (decrees, the agenda, a pinned line), but it cannot carry colors, links, textures or line breaks. |
| S8B-7 | Info | **Global error handler.** The addon wraps the game's error handler (`Bootstrap.lua:42-76`). It records up to 40 of its own errors for the local bug report, then always passes every error to the previous handler. It hides nothing. Verified. |
| S8B-8 | Info | **Author-only tools** are gated by the author's character name, "Faladoriel Skylance" (`Workshop.lua:99-116`): a "view as" preview and a photo mode that hides the interface. For anyone else they do nothing. |
| S8B-9 | Info | **Performance.** The main window rebuilds its tab every 5 seconds only while it is open (`UI.lua:1066-1071`). Redraws caused by network traffic are throttled to one per half second. Borders and nameplates react to events and never run every frame. |

Part B result: **clean.** The visible effects on the game's own interface are cosmetic, and each one can be turned off.

## Stage 9: Olympus Link and the web side

Files read in full by a delegated reviewer: `Link.lua`, every file in `web/public/` (skimming
the vendored `jsQR.js`), `web/worker/*`, `web/tools/read-inbox.mjs`, the web docs,
`scripts/link-keys.py`, `scripts/council-sign.py` and `pages.yml`. I confirmed the placeholder
key myself (`Link.lua:75`).

| ID | Severity | Finding |
|---|---|---|
| S9-1 | Info | **Olympus Link is switched off in this release.** The bot's public key is a placeholder (`Link.lua:75`, verified), and so are the page's backend address and Discord app ID (`web/public/config.js:9`, `12`). Every code is refused. A player who never types `/oly discord` sends nothing. |
| S9-2 | Info | **When it goes live, nothing leaves before you click Accept.** The QR code opens `https://dnl-gentile.github.io/olympus-addon/#b=...`. The proof sits after the `#`, which browsers do not send to servers. It holds your character, realm, guild and faction, a nonce, a tag, and the confirmers' names, signatures and certificates. It holds no secret and no Discord username. |
| S9-3 | Low | **Your proof can go to any High Councillor in watcher mode**, not only the bot's own watcher (`Link.lua:726`, `1187`). It then sits in that person's saved-variables file, and your addon reports it as delivered. |
| S9-4 | Low | **The Link page loads Google Fonts** (`web/public/index.html:17`), so Google sees your IP address and browser. The README's privacy section does not mention it. |
| S9-5 | Low | **Discord sign-in uses the older implicit-grant flow** with the `identify` scope only. The token goes to the bot's Worker and is not revoked after use. A state check, removal from the address bar and tab-only storage reduce the risk. |
| S9-6 | Low | **A confirmer's private key is stored in plain text** in `OlympusDB` (`Link.lua:1500`), readable by any other installed addon and by anyone with the file. This only applies if you become a confirmer. |
| S9-7 | Info | **The web page is tidy.** Its Content-Security-Policy allows scripts from its own origin only. It contacts only the bot's Worker, discord.com for sign-in, and Google Fonts. It has no analytics, uses the camera only on a click, and keeps sign-in state for the tab only. |
| S9-8 | Info | **The reference Worker** stores the link between your Discord ID and your character. It keeps IP addresses only as a truncated keyed hash, for 60 seconds of rate limiting. The live Worker is run by the bot's keeper and is not in the repository, so its logging cannot be checked. |
| S9-9 | Info | **No committed secrets.** Test seeds are derived from public labels, and the certificate-authority key and RSA modulus are public keys. |

Stage 9 result: **dormant in this release, and sensible in design.** If it goes live, linking
shares your character-to-Discord mapping with the bot's operator, which is the point of the feature.

## Stage 10: Bundled libraries

A delegated reviewer fetched each library's upstream source into `/tmp/olympus-audit/upstream/`
and compared it with the vendored copy.

| Library | Version | Compared with | Result |
|---|---|---|---|
| LibStub | MINOR 2 | Ace3 `Release-r1403` | Same code. Only the header comment differs. The file also embeds a byte-identical copy of CallbackHandler rev 8. |
| CallbackHandler-1.0 | MINOR 8 | Ace3 `Release-r1403` | Byte-identical, license too. |
| HereBeDragons-2.0 and -Pins-2.0 | MINOR 33 and 17 | Nevcairiel/HereBeDragons `2.17-release` | Both byte-identical. |
| QREncode (luaqrcode) | 2012-2020, BSD-3 | speedata/luaqrcode `5a9c34b`, the closest revision (no tags) | Only the changes the file itself declares (lines 30-35): the game's `bit.bxor`, an optional pause callback, a removed test export, and registration on the addon's namespace. |

The libraries contain no code loading, chat or addon-message calls, or hooks. `Map.lua:576-598`
wraps one function of the map-pins library, and only when Olympus's own copy is loaded. This is benign.

The locale files are plain string tables. `ns.Locale` accepts a translation only if its format
and escape codes match the English line (`Locales.lua:3086-3093`), so a translation cannot add
links. There are no URLs or hyperlink codes in any locale.

Stage 10 result: **clean.**

## Stage 11: Developer files that do not ship

None of these are in the release zip. They matter only if you run them yourself.

- `scripts/check.sh`, `lint-globals.sh`, `gamepad-audit.lua`, `forever-pins.lua`, `curseforge-*.lua`, `answers.lua`, `make-borders.py`: local build, lint and generation tools. No network.
- `scripts/package.sh` zips `Olympus/` into `dist/`.
- `scripts/deploy.sh` and `logs.sh` use **your own** SSH access to a Windows PC. `deploy.sh` deletes and replaces the remote `AddOns/Olympus` folder.
- `scripts/watch-deploy.sh` polls `git fetch origin` every 20 seconds and runs `deploy.sh` from the fetched commit. That means it **runs downloaded code**. This is a risk only for a developer who runs it.
- `scripts/forever-api.lua` runs Blizzard's API documentation files from a local client extract, unsandboxed. Developer only.
- `scripts/council-sign.py` and `link-keys.py` create and use the author's RSA and Ed25519 signing keys, kept in `~/.olympus/` or `dist/` with mode 0600. No private key is in the repository or the release.
- `tests/run.lua` and `tests/gamepad.lua` use `io.popen` only to list files in `Olympus/`. No network.
- CI (`.github/workflows/`): read-only permissions (plus the standard Pages permissions), no secrets referenced, and every third-party action pinned by commit SHA.

Stage 11 result: **nothing harmful.** Do not run `watch-deploy.sh` or `deploy.sh`. The audit itself ran none of these.

## Stage 12: Distribution check

| Item | Result |
|---|---|
| Upstream tag `v1.1.5` | Annotated tag `8a8f226a`, pointing to `5825ed4`, the audited commit. All 39 local tags match the remote. |
| GitHub release asset | `Olympus-1.1.5.zip`, 1,045,494 bytes |
| SHA-256 | `386dc19579803f220d65a76b4d3728af9d3644009ef6193341f0d370152a710f` (verified myself, and equal to GitHub's recorded digest) |
| Contents | 83 entries, all under `Olympus/`, no `..` paths or symlinks. `diff -r` against the audited `Olympus/` folder shows **no differences** (verified myself). |
| CurseForge copy | **Not verified.** CurseForge returned HTTP 403 (a Cloudflare challenge) to automated access. |

Stage 12 result: **the GitHub release is exactly the audited code.** The CurseForge download
could not be checked. To be sure of what you run, install the GitHub zip by hand, or compare
a CurseForge download against the hash above:

```sh
sha256sum Olympus-1.1.5.zip
```

## Stage 13: Verdict and recommendations

### Verdict

**No malicious code was found. Overall risk is low, and the addon is safe to install with the
settings below.**

| Severity | Count |
|---|---|
| Critical | 0 |
| High | 0 |
| Medium | 2 |
| Low | 30 |
| Info | 58 |

- **Your machine: no risk from the addon.** It runs in Blizzard's Lua 5.1 sandbox with no file,
  network or process access. It never turns text into code. Its only off-game feature, Olympus
  Link, is switched off in 1.1.5. The GitHub release zip is byte-for-byte the audited code.
- **Your account: nothing harmful happens without your click.** Nothing moves gold or items,
  and nothing sends visible chat on its own. The only automatic group actions belong to a layer
  hop you ask for, or to layer help you opt into. Guild invites and removals need an officer's
  click and a confirmation.
- **Your privacy: personal sharing is off until you say yes.** The exceptions are your
  addon version and a hello to your guild. Watch two things. The census names guild leaders,
  officers and the five highest-level members to the whole channel. And one Yes covers every
  character on the account.
- **Your screen: others can show you text, but nothing more.** Leaders, and a few colluding
  strangers while the channel is unsealed, can show raid warnings and popups with plain text.
  They cannot add links, run code or act as you.

The two Medium findings are S5-1 and S6-1:

1. **S5-1:** the census broadcasts guild leaders', officers' and top-level members' details by default, with no opt-out. It is documented, and it affects you whether or not you install, if you hold one of those positions.
2. **S6-1:** while the channel is unsealed, characters who fake Captain rank through the census can push raid-warning text with sound. Sealing the channel stops this.

### Answers to the questions in CLAUDE.md

- **Which Lua interpreter?** Blizzard's embedded, sandboxed Lua 5.1, inside the modern retail
  ("Mainline") client that WoW: Forever runs on. Stage 2 has the evidence and a two-line
  in-game check.
- **Anything malicious?** No. There is no obfuscation, no code loading, no hidden privilege, no
  backdoor and no off-game data path in this release.

### Recommendations

1. **Install the GitHub release zip by hand** and check its SHA-256 against
   `386dc19579803f220d65a76b4d3728af9d3644009ef6193341f0d370152a710f`. The CurseForge copy could
   not be checked. If you use the CurseForge app, compare its zip with this hash.
2. **Pin the version.** The project has shipped several releases a day. Turn off auto-update,
   and re-check each new version before installing it, using the recipe below.
3. **Answer the first-open page deliberately.** It appears about 45 seconds after login, or
   with `/oly privacy`. The answers apply to every character on the account. A privacy-minded
   baseline:
   - Zone and layer: No, unless you want layer help.
   - Layer help: No. Never choose "Always invite".
   - Royal Inspection: No.
   - The author's roll call: your choice. It sends only your version and client.
   - Olympus chats: Yes if you want them, but treat them as public.
   - Patrol findings (officers only): your choice.
   - Leave the map dot (`/oly share`) off.
4. **Optional switches.** `/oly chatmarks off` keeps the marks out of your normal chat.
   `/oly filter shared off` stops leadership's word list from hiding Olympus lines.
   `/oly nocontact on` stops the Join screen from pointing recruits to you.
5. **Treat Olympus text as unverified.** Strangers can write decrees, polls and pinned lines
   on an unsealed channel. Never act on a URL or a gold request in them. A mark next to a
   name does not prove identity.
6. **Read the amount before you pay dues.** The dues button fills in an amount the King or a
   Steward chose, up to 1,000 gold. The recipient is fixed.
7. **If you become an officer, seal the channel** with `/oly key <secret>`. That shuts strangers
   out of the decrees and the chats.
8. **Do not run the repository's developer scripts.** None are needed to play, and
   `watch-deploy.sh` runs downloaded code.
9. **Olympus Link is inert in 1.1.5.** If a later version switches it on, linking shares your
   character-to-Discord mapping with the bot's operator, and the page loads Google Fonts.

### The 30 Low findings, grouped by what to do about them

Added after the audit, in answer to a follow-up question. None of the Low findings is a reason
to skip the addon. Here, Low means limited impact: nothing runs code, nothing moves gold or items
without your click, and nothing leaves the game. The findings fall into three groups. Some call
for a setting or a habit. Most are quirks, bugs, or docs that promise more than the code does. A
few do not apply yet.

#### Worth a setting or a habit

- **Privacy answers** apply to your whole account. A Yes on one character applies to every character you have in an Olympus guild, and the README never says so. Answer as if for your most private character. (S7A-1)
- **The Enter key** belongs to Olympus while its Chat tab is open. Lines you type then go to the Olympus channel, which may be public, instead of guild chat or a whisper. Only your first line on each channel asks for confirmation. (S8A-3)
- **"Always invite"** for layer help sends party invites to anyone who asks on the channel, Olympus member or not. Leave it off. (S7A-2)
- **The dues button** fills in an amount chosen by the King or a Steward. The README says changes wait a week, but a dishonest one could change this week's amount. The recipient is fixed, so read the amount before pressing Send. (S4-3, S7B-2)
- **Marks before names** are cheap to earn: a made-up guild name plus one self-report is enough. A mark in Trade chat proves nothing. (S8B-2)
- **Item links in Olympus chat** can show a fake name or color. Hover over one to see what it really links to. (S8A-1)
- **Marks in your normal chat** are on by default. They only add a small image before members' names, and you can turn them off. (S8B-1)
- **Leadership's shared word filter** is on by default. It only hides Olympus text, never your normal chat, and you can opt out. (S6-4)
- **One Hand or High Councillor** can hide almost anyone from Olympus screens for a month. It never touches normal chat, but the check is the sender's name alone, with no signature. (S6-3)
- **During a layer hop you started,** the addon accepts an invite for you if it comes from someone your client asked and whose rank it trusts. A rank faked through the census can pass that check. (S4-4)
- **A rare timing case** could make you leave a friend's group partway through a hop. If you accept their invite before the game shows the group's member names, the addon may take it for the hop group. This is inferred from the code and was not observed in game. (S7A-3)
- **Linking alts** leaves a trail. Alt claims carry a timestamp shared across your account, which can tie characters together even after you unlink one. (S7A-4)
- **Pasting a backup** can change your settings once you confirm. It never touches your privacy answers or the channel key. Still, never paste backup text someone else hands you. (S7B-6)

The switches mentioned above:

```
/oly privacy
/oly chatmarks off
/oly filter shared off
/oly layerauto off
```

#### Quirks, bugs and docs that overpromise

- **Channel-owner powers** do get used, despite what SECURITY.md says. If the game makes you the channel owner, your client resets the password, turns moderation off, and lifts bans and mutes. (S5-2)
- **Whoever owns an unsealed channel** can lock it with a password. The addon keeps retrying, and the game may ask you for the password. Press Cancel. (S5-6)
- **A crafted census report** could briefly freeze clients through slow text matching. It is unmeasured, probably under a second, and each sender is rate-limited. (S5-3)
- **Fake guild reports** can grow your saved-variables file. Each one is kept for a week, and nothing caps how many there can be. (S5-5)
- **The channel key** comes from weak randomness. That barely matters, because the channel is not encrypted anyway. (S6-5)
- **Anyone can ask your addon** whether you run Olympus and which version. Players you have on ignore get no answer. (S5-4)
- **Census zone counts** include members who said No, because only the reporting member's choice controls them. In a small guild a count can point at one person. The README says nothing relays another player's zone, which is not quite true. (S7A-5)
- **Dues payments** feed a removal tool for officers. The Treasurer's client tells your guild's officers who paid, and mail the Treasurer has not opened yet counts as unpaid. This happens whether or not you install. (S7B-3)
- **Two taint risks** come from Olympus's popups and from clicking a Board card. Taint is Blizzard's flag on addon-touched code, and either can occasionally block a protected Blizzard action until you reload. The Board click also opens the game's chat box, which the README says Olympus never does. (S8B-3, S8A-2)
- **Bank tab names** sent by a treasury keeper can carry colors or icons. This is cosmetic. (S7B-7)

Several of these are places where the docs promise more than the code does. That fits a project
shipping several releases a day, and it does not suggest deception. It does mean the code, not
the README, should be the reference when re-checking an update.

#### Not your concern yet

- **The Olympus Link findings** cover a feature that is switched off in this release. If it goes live, your proof could be held by any High Councillor in watcher mode, and the page loads Google Fonts. Discord sign-in would also hand a login token to the bot's server, and a confirmer's private key would sit in plain text on disk. (S9-3, S9-4, S9-5, S9-6)
- **The Horde King's identity** is trusted on first use. This only matters on the Horde side. (S6-2)

If you keep only a few habits: answer the privacy page as if for your most private character,
and leave Always invite off. Read the dues amount before sending. Treat marks, links and raid
warnings as unverified.

### Verify it yourself in game

```
/run print(_VERSION)
/run print(io, require, dofile, loadfile, os and os.execute)
/oly privacy
/oly status
```

The first line should print `Lua 5.1`, and the second five `nil`s.

### How to re-check an update

```sh
cd /home/darkmage/src/olympus-addon
git fetch --tags
git log --oneline v1.1.5..origin/main
git diff --stat v1.1.5 origin/main -- Olympus/
git diff v1.1.5 origin/main -- Olympus/ | grep -nE '^\+.*(loadstring|RunScript|setfenv|getfenv|SendChatMessage|SendMail|SetSendMailMoney|SetTradeMoney|AcceptTrade|PickupContainerItem|DeleteCursorItem|UseContainerItem|InviteUnit|AcceptGroup|GuildUninvite|SetCVar|SetBinding|CreateMacro|ReloadUI|LINK_BACKEND_KEYS|LINK_COUNCIL_AUTHORITY)'
```

Any hit deserves a manual look. A new value in `LINK_BACKEND_KEYS` means Olympus Link went live.

### Limits of this audit

- It covers version 1.1.5 at commit `5825ed4` only.
- I did stages 1 to 4 myself. The line-by-line reads behind stages 5 to 12 were done by eight
  delegated reviewers. I re-read the code behind both Medium findings and the most consequential
  Low ones (dues, layer hop, consent storage, chat marks, census, decrees, the error handler, the
  release zip), and every claim I checked was accurate. I did not personally read every line.
- Nothing was executed. In-game behavior such as taint, Blizzard's restrictions and the hop
  timing case is inferred from the code.
- The CurseForge download and the live Link Worker could not be checked.

### Session summary

All stages, 0 to 13, are complete. A future session could re-audit the next release with the
recipe above, or confirm the layer-hop and taint behaviors in game.

After the audit, a follow-up question added the grouped walkthrough of the Low findings
above, under "The 30 Low findings, grouped by what to do about them".
