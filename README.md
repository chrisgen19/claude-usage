# claude-usage

[![ci](https://github.com/chrisgen19/claude-usage/actions/workflows/ci.yml/badge.svg)](https://github.com/chrisgen19/claude-usage/actions/workflows/ci.yml)

The 5-hour and 7-day limits of every Claude Code profile you run, of Codex,
and of Antigravity, on one screen, without opening a session in each.

A single bash script. Needs `curl` and `jq`, nothing else, and `agy` for
Antigravity.

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

 CODEX  alex  plus  updated 12:31
   5h      ███████▌░░░░░│░░░░░░░░░░░░░░  27%  resets 15:02            in 2h31m
   7d      ████████████████│██░░░░░░░░░  68%  resets Mon 28 Sep 08:31 in 2d20h

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

Codex has the same two windows and the same problem: `/status` shows them, but
only inside a session. So its profiles sit on the same screen. So does
Antigravity, whose `/usage` has its own 5-hour and weekly limits.

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
5. **Adds Codex.** `~/.codex`, `$CODEX_HOME` and any `~/.codex-*/` Codex has
   used each get a `CODEX` block. The login in `auth.json` goes to the
   endpoint Codex asks for its own limits. See [Codex](#codex).
6. **Adds Antigravity.** When `agy` is on your `PATH` and signed in, an `AGY`
   block shows what agy's own `/usage` says. See [Antigravity](#antigravity).

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

## Codex

A Codex profile is a `CODEX_HOME`: `~/.codex` is labelled `CODEX`, and
`~/.codex-work` is labelled `CODEX-WORK`. Its 5h and 7d rows are Codex's two
windows, and a plan that only has a weekly limit shows just the 7d row.

**When the live read fails**, whether the login has expired, the connection is
down, or `chatgpt.com` turns the request away, a Codex profile has a second
source. Every Codex session logs the limits it was told on each turn, so
`claude-usage` takes the newest from the session logs when it is newer than
the reading it already has. It says so: `as of 09:31 (3h ago) from Codex's
session log`. Only a reading taken after the login was saved is used, since
an older one may belong to an account that has since logged out. A token
refresh saves the login too, so readings from before one wait for Codex's
next turn.

**An API key login** (`codex login --with-api-key`) is billed per token and
has no plan limits, and says so.

**Not read:** a Codex login kept in the OS keyring instead of `auth.json`.
Its session logs still are.

## Antigravity

Antigravity's limits come from its CLI, `agy`, which `claude-usage` runs as
`agy -p /usage --output-format json`: the panel `/usage` shows in a session,
printed once. It is a slash command, not a prompt, and takes agy a few
seconds. agy keeps one login, in `~/.gemini/antigravity-cli`, so there is one
`AGY` block at most.

```
 AGY  alex  updated 09:34
   gemini 5h │░░░░░░░░░░░░░░░░░░░░░░░░░   0%  resets 14:34            in 4h59m
   gemini 7d │░░░░░░░░░░░░░░░░░░░░░░░░░   0%  resets Mon 26 Oct 09:34 in 6d23h
   claude 5h │░░░░░░░░░░░░░░░░░░░░░░░░░   0%  resets 14:34            in 4h59m
   claude 7d │░░░░░░░░░░░░░░░░░░░░░░░░░   0%  resets Mon 26 Oct 09:34 in 6d23h
   argon 7d  ██│░░░░░░░░░░░░░░░░░░░░░░░  11%  resets Sun 25 Oct 15:18 in 6d5h
```

Antigravity's quota is shared by groups of models, and each group has a
5-hour and a weekly limit. The rows are:

| Row | Group |
| --- | --- |
| `gemini 5h`, `gemini 7d` | Gemini models |
| `claude 5h`, `claude 7d` | Claude and GPT models (Claude Opus, Sonnet and GPT-OSS) |
| `argon 7d` | Gemini 4 Argon, which agy reports with a weekly limit only |

agy reports how much is **left**; the bars show how much is used, like every
other row. A group agy does not report has no row. A 5h window that has not
started yet resets 5 hours from now and stays at 0%.

**Why agy and not its endpoint.** agy reads its limits from a private Google
endpoint, and Google has disabled accounts whose Antigravity login was used by
other tools. Running agy keeps the login inside agy: `claude-usage` never reads
the access or refresh token, only the email in the `id_token` (for the name it
shows and to file the reading under the right account) and whether there is a
refresh token at all. agy renews its own login when it runs, so this block
does not expire the way an idle Claude or Codex login does.

**When agy cannot answer**, the block keeps its last reading with the reason:
agy's own error, `agy could not sign in` when its login has lapsed (run `agy`
to sign in again), or `timed out` after `CLAUDE_USAGE_AGY_TIMEOUT` seconds.
With no login at all agy would wait a minute for one to be typed in, so a
login without a refresh token is reported, and agy is not run. agy is run in
an empty temporary directory, so it never reads a project, with nothing on
stdin, and with `BROWSER=true` in case a lapsed login would try to open a
sign-in page.

**Not read:** an agy login kept somewhere other than
`~/.gemini/antigravity-cli/antigravity-oauth-token`, such as the macOS
Keychain, and the Antigravity IDE's own login. Quota belongs to the Google
account, so an IDE signed into the same account as agy should show the same
numbers, but that has not been checked.

## Logins

A login is **read, never refreshed**. Refreshing an OAuth login rotates the
token, and rewriting it underneath a running Claude Code session can sign that
session out. So when a login expires (after a few hours unused), the profile
says so and keeps its last reading until you next open Claude Code with it,
which renews the login the normal way. A Codex login is treated the same way,
and is renewed by opening Codex. agy's login is not read at all: agy is run
instead, and renews its own. See [Antigravity](#antigravity).

The token never leaves memory except to go to `api.anthropic.com`, or to
`chatgpt.com` for a Codex login:

- It reaches `curl` on stdin (`-H @-`), never on a command line, so `ps` cannot
  show it.
- The cache in `~/.cache/claude-usage/` holds numbers only: when each reading
  was taken, a checksum standing in for the account, and the readings. Files
  are named for the account and organisation ids from `.claude.json`, or a
  Codex login's user and workspace ids, which are identifiers, not secrets.
  The workspace id goes to `chatgpt.com` in a header, beside the token.
- Nothing is printed but the account's user name (the part before the `@`) and
  plan.

The self test asserts all three.

## Notes and caveats

- **The usage endpoint is not a public API.** It is what Claude Code's own
  `/usage` calls, and it could change without notice. If it does, profiles show
  `unexpected answer from the usage endpoint` rather than wrong numbers.
- **agy's JSON output is not a documented format.** It is what
  `agy -p /usage --output-format json` prints, tried against agy 1.4.2. If it
  changes, the `AGY` block shows `unexpected answer from agy /usage` rather
  than wrong numbers.
- **The Codex endpoint is not a public API either.**
  `chatgpt.com/backend-api/wham/usage` is what Codex's own client asks for
  its limits. The self test covers it through a stub, but it has not been
  tried against a live ChatGPT login yet. How often it may be read is not
  known, so it keeps to the same one read a minute and the same waits after
  a 429. `chatgpt.com` can also turn a request away with a page of its own
  before it reaches the endpoint. That shows as `chatgpt.com turned the
  request away (403)`, not as a rejected login, and the session logs stand in.
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
| `CLAUDE_USAGE_CODEX_DIRS` | Colon-separated Codex homes to watch. Set but empty, no Codex profiles at all |
| `CODEX_HOME` | A Codex home to watch besides `~/.codex` and `~/.codex-*/` |
| `CLAUDE_USAGE_AGY` | The agy command to run (default `agy`). Set but empty, no Antigravity at all |
| `CLAUDE_USAGE_AGY_TIMEOUT` | Seconds agy gets to answer (default 30) |
| `CLAUDE_USAGE_ASCII` | `1` draws with `#` and `.` instead of block glyphs |
| `NO_COLOR` | `1` turns colour off |
| `XDG_CACHE_HOME` | Last readings live in `$XDG_CACHE_HOME/claude-usage` (default `~/.cache`) |

## Development

```bash
shellcheck --severity=style claude-usage scripts/bump scripts/selftest
scripts/selftest
```

`scripts/selftest` is the whole test suite and runs anywhere, offline. `curl`
and `agy` are replaced by stubs on `PATH` that answer by token, and every
profile is a throwaway directory. Besides the readings themselves (offsets and
fractional seconds in reset times, plans with extra weekly limits, Codex's answers and
session logs, agy's `/usage` groups, every failure path), it asserts that a token never reaches a command line, the cache or the screen, and
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
