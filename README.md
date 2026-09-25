# claude-usage

[![ci](https://github.com/chrisgen19/claude-usage/actions/workflows/ci.yml/badge.svg)](https://github.com/chrisgen19/claude-usage/actions/workflows/ci.yml)

The 5-hour and 7-day limits of every Claude Code profile you run, on one screen,
without opening a session in each.

A single bash script. Needs `curl` and `jq`, nothing else.

```
 CLAUDE USAGE  Fri 25 Sep 12:31:41

 DEV  alex  pro  updated 12:31
   5h      █▉░░░░░░░░░░░░░░░░│░░░░░░░░░   7%  resets 14:13            in 1h42m
   7d      ██│▎░░░░░░░░░░░░░░░░░░░░░░░░  12%  resets Fri 02 Oct 00:31 in 6d12h

 PERSONAL  alex.home  max 5x  updated 12:31
   5h      ████████████▌░░░░░░│░░░░░░░░  45%  resets 14:03            in 1h32m
   7d      █████████████│██▊░░░░░░░░░░░  60%  resets Tue 29 Sep 03:31 in 3d15h
   7d opus ██████▏░░░░░░│░░░░░░░░░░░░░░  22%  resets Tue 29 Sep 03:31 in 3d15h

 WORK  alex.w  team  as of 09:31 (3h ago)
   ! login expired 1h ago: open Claude Code with ~/.claude-work to renew it
   5h      ░░░░░░░░░░░░░░░░░░░░░░░░░░░░   0%  reset at 12:21
   7d      ██████████████████████░│░░░░  79%  resets Sat 26 Sep 16:51 in 1d4h

 │ time gone in each window: usage past it is ahead of pace
 r refresh  q quit
```

## Why

Running more than one Claude account usually means one `CLAUDE_CONFIG_DIR` per
account: `~/.claude-work`, `~/.claude-personal`, and so on. Each profile's
statusline can show its own rate limits, but only for itself and only while a
session is open in it, because Claude Code hands those numbers to the
statusline rather than the statusline asking for them. Finding out whether the
work account has headroom before you switch to it means opening it and running
`/usage`.

`claude-usage` asks for all of them at once, and keeps asking.

## How it works

1. **Finds your profiles.** Every `~/.claude-*/` directory Claude Code has
   written its state into. If there are none, plain `~/.claude`.
   `CLAUDE_USAGE_DIRS` overrides the search.
2. **Reads each login** from `.credentials.json` in that directory, or from the
   Keychain on macOS, where Claude Code keeps it instead.
3. **Asks the usage endpoint** behind `/usage` for each profile, all in
   parallel. These are account-wide numbers: use on another machine, or in
   claude.ai, counts too.
4. **Keeps the last good reading** per profile, so an expired login or a
   dropped connection still shows something, marked with its age.

## Install

Needs **bash 4.2+**, **curl 7.55+** and **jq**. macOS ships bash 3.2, so install
a current one first:

```bash
brew install bash jq        # macOS only
```

```bash
curl -fsSL https://raw.githubusercontent.com/chrisgen19/claude-usage/main/claude-usage -o ~/.local/bin/claude-usage
chmod +x ~/.local/bin/claude-usage
```

Make sure `~/.local/bin` is on your `PATH`. Or clone and symlink:

```bash
git clone https://github.com/chrisgen19/claude-usage.git
ln -s "$PWD/claude-usage/claude-usage" ~/.local/bin/claude-usage
```

## Commands

| Command | What it does |
| --- | --- |
| `claude-usage` | Live view in a terminal; one snapshot when piped |
| `claude-usage watch [secs]` | Live view, refreshing every `secs` (default 60, at least 60) |
| `claude-usage once` | Print one snapshot and exit |
| `claude-usage profiles` | The profiles found and whether each login is still valid. No network |
| `claude-usage help` | All of the above |

Keys in the live view: `r` refreshes now, `q` quits. `r` keeps to the rate
limit below, so an account read under a minute ago is not asked again.

## Reading it

**5h** and **7d** are the same rolling limits `/usage` shows. Plans that have a
separate weekly Opus or Sonnet limit get a `7d opus` or `7d sonnet` row too.
Percentages turn yellow at 70% and red at 90%, like the statusline.

**The `│` marker** is how far into that window you are. A bar that has filled
past its marker is using the window faster than time is passing, so at that
rate it runs out before it resets. In the sample above, WORK's weekly bar is
just short of its marker: on pace, with little room to spare.

**A profile that could not be read** keeps its last reading, labelled
`as of 09:31 (3h ago)`, with the reason underneath. A window whose reset time
has passed since then shows `0%  reset at ...`, because it restarted from zero.
If you have used that account elsewhere since, the real figure may be higher.
A reading belongs to the account and organisation it was read for (one email
can hold a Pro plan and a Team seat, each with its own limits). If a profile is
logged into another, the old reading is dropped rather than shown under the
new name.

## Logins

A login is **read, never refreshed**. Refreshing an OAuth login rotates the
token, and rewriting it underneath a running Claude Code session can sign that
session out. So when a login expires (after a few hours unused), the profile
says so and keeps its last reading until you next open Claude Code with it,
which renews the login the normal way.

The token never leaves memory except to go to `api.anthropic.com`:

- It reaches `curl` on stdin (`-H @-`), never on a command line, so `ps` cannot
  show it.
- The cache in `~/.cache/claude-usage/` holds numbers only: when each reading
  was taken, a checksum standing in for the account, and the readings. Files
  are named for the account and organisation ids from `.claude.json`, which
  are identifiers, not secrets.
- Nothing is printed but the account's user name (the part before the `@`) and
  plan.

The self test asserts all three.

## Notes and caveats

- **The usage endpoint is not a public API.** It is what Claude Code's own
  `/usage` calls, and it could change without notice. If it does, profiles show
  `unexpected answer from the usage endpoint` rather than wrong numbers.
- **It is rate-limited,** to about one read a minute per account; a second one
  gets a 429. So a reading under a minute old, taken by any `claude-usage` on
  this machine, is reused instead of asked for again, which lets a live view
  and a `once` run, or two live views, share the minute; so do two profiles
  logged into the same account. If two copies still ask in the same moment,
  the one that gets the 429 takes the other's reading instead of waiting.
  After a real 429 that account waits 2 minutes, then 4, then 5, and the wait
  is kept beside the cache so every `claude-usage` here keeps to it. The endpoint does send
  `retry-after`, but has said `0` and then refused the retry, so it only
  counts when it asks for longer. A copy on another machine cannot see any of
  this, so it may still collide; the wait is what keeps that from repeating.
- **macOS reads the Keychain.** Claude Code 2.1 names the item for how it was
  started: `Claude Code-credentials` with `CLAUDE_CONFIG_DIR` unset, otherwise
  that plus `-` and the first 8 hex characters of `sha256(CLAUDE_CONFIG_DIR)`.
  `~/.claude` tries both, every other profile the second. The first read may
  ask for Keychain access; choose *Always Allow*. The self test covers this
  through a stand-in `security`, but it has not been tried on a Mac with real
  logins yet.
- **The timezone is yours.** Reset times are shown in local time, using bash's
  own `strftime`, so GNU and BSD `date` never disagree.

## Environment

| Variable | Effect |
| --- | --- |
| `CLAUDE_USAGE_DIRS` | Colon-separated config dirs to watch, e.g. `~/.claude-work:~/.claude-personal` |
| `CLAUDE_USAGE_ASCII` | `1` draws with `#` and `.` instead of block glyphs |
| `NO_COLOR` | `1` turns colour off |
| `XDG_CACHE_HOME` | Last readings live in `$XDG_CACHE_HOME/claude-usage` (default `~/.cache`) |

## Development

```bash
shellcheck --severity=style claude-usage scripts/bump scripts/selftest
scripts/selftest
```

`scripts/selftest` is the whole test suite and runs anywhere, offline. `curl`
is replaced by a stub on `PATH` that answers by token, and every profile is a
throwaway directory. Besides the readings themselves (offsets and fractional
seconds in reset times, plans with extra weekly limits, every failure path), it
asserts that a token never reaches a command line, the cache or the screen, and
that an expired login is never sent at all.

Both the ci and release workflows run that same script, so a tagged release
cannot publish a build that leaks a login.

### Releasing

```bash
scripts/bump patch      # 0.1.0 -> 0.1.1, commits and tags
git push origin main && git push origin v0.1.1
```

Pushing a `v*` tag builds a GitHub Release. The release workflow refuses any tag
that disagrees with `VERSION` inside the script.

## License

MIT. See [LICENSE](LICENSE).
