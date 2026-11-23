+++
title = "Tickcli"
description = "Command-line client for Tickbuddy"
weight = 2
+++

This page describes Tickcli 0.2.0, released 22 November 2026.

## NAME

`tickcli` - the command-line client for Tickbuddy, the daily habit and event tracker for Nextcloud. Pronounced "tickly".

## SYNOPSIS

```
tickcli [-d date] [--private] [--archived]
tickcli login <url> [--paste] [--qr] [--insecure-http]
tickcli logout [--local]
tickcli status

tickcli tick <track> [-d date] [note options]
tickcli untick <track> [-d date] [note options]
tickcli set <track> <value> [-d date] [note options]
tickcli add <track> [n] [-d date] [note options]
tickcli note <track> (<text> [--append [--force-duplicate]] | --clear) [-d date]

tickcli tracks [--private] [--archived]
tickcli ticks [track] [-d date | --from date [--to date] | --year YYYY]
tickcli week [--private]
tickcli notes [--from date] [--to date]
tickcli stats [track] [--from date] [--to date] [--year YYYY]
tickcli export [--private]

tickcli help [command]
tickcli --version
```

The note options are `--note text [--append [--force-duplicate]]` or `--append-note text [--force-duplicate]`.

## DESCRIPTION

Tickcli reads and writes your Tickbuddy tracks from the command line. Use it from scripts, cron jobs and home automation to log what happens ("nightly backup finished"), or instead of the web app or Tickdroid when you're at a terminal. It is also the building block of TickLLMe, which signs in with the same credentials.

Run without a command, Tickcli shows today's tracks with their values and notes.

### Tracks

Name a track by its exact name, ignoring case (`exercise` for "Exercise"), or by its id as `#n` (`#12`). `tickcli tracks` lists the ids. When you're at a terminal, a unique prefix also works (`cups` for "Cups of coffee"). Scripts always need the full name or the id, so a new track can never be hit by accident. Quote a name with spaces: `"Nightly backup"`.

Archived and private tracks can always be named directly. Lists leave them out unless you add `--archived` or `--private`.

Tickbuddy has four types of track, and each takes its own commands:

- **Yes/No:** `tick` and `untick`.
- **Counter:** a whole number. `set` sets it and `add` adds to it.
- **Number:** a reading such as 10.4 km, with the track's unit and decimal places. `set` sets it.
- **Choice:** one of the track's options, such as Happy. `set` sets it.

Using the wrong command for a track's type is bad usage (exit status 2). `tickcli tracks` shows each number track's unit and decimal places and each choice track's options.

### Dates

Most commands work on today, the machine's local date, not the server's. Use `-d` to choose another day:

- `YYYY-MM-DD`, such as `2026-11-23`
- `today` or `yesterday`
- `-N` for N days ago, such as `-2`

The same forms work for `--from` and `--to`. A day in the future is refused.

### Numbers and choices

- A number uses a point as the decimal separator and no grouping: `10.4`, not `10,4` or `1,204.5`.
- The server rounds a number to the track's decimal places, half away from zero. On a track with 1 decimal place, `tickcli set Run 10.46` stores 10.5. Tickcli shows the value the server stored.
- A number below 0 needs a track that allows negative values. `tickcli tracks` says which do.
- A choice's option is its label, ignoring case (`happy`), or `'#2'` for the second option. Quote `'#2'`, since `#` starts a comment in a shell. A bare `2` is always a label, never a place.
- `""` clears a number or choice day. An empty variable in a script does the same: `tickcli set Run "$km"` with `$km` empty leaves Run without a value rather than failing.

### Notes

Each track can have a note for each day, of up to 500 characters. `tick`, `untick`, `set` and `add` can set the note in the same request, and `note` sets it on its own.

- `--note text` replaces the day's note. `--note ""` deletes it.
- `--append-note text`, or `--note text --append`, adds the text to the note on a new line instead.
- When appending, text the note already holds is skipped, so running the same command again changes nothing. `--force-duplicate` appends it anyway.

### Retries

A request that is safe to repeat is retried up to 3 times, after 1, 2 and 4 seconds, when the server can't be reached. `--retries n` changes how many times. Two kinds of request are never retried, because a retry could log the same thing twice if the server got the first one and only the answer was lost:

- `add`. When `add` fails with exit status 5, check with `tickcli` before adding again.
- An append with `--force-duplicate`.

### Installation

Download the binary for your system from the [releases page](https://github.com/martinhammer/tickcli/releases), make it executable and put it on your `PATH`. Binaries are published for Linux (amd64, arm64, armv7), macOS (amd64, arm64) and Windows (amd64). With Go installed, you can instead run:

```
go install github.com/martinhammer/tickcli/cmd/tickcli@latest
```

macOS may block the downloaded binary because it isn't signed. Run `xattr -d com.apple.quarantine tickcli` once, or allow it under System Settings → Privacy & Security.

## COMMANDS

### tickcli

Show today's tracks with their values and notes. Use `-d` for another day, and `--private` and `--archived` to include those tracks.

### login *url*

Sign in to a Nextcloud that has Tickbuddy. A URL without a scheme gets `https://`. Plain `http://` is refused except for `localhost` and private addresses.

By default, Tickcli uses Nextcloud's [Login Flow v2](https://docs.nextcloud.com/server/stable/developer_manual/client_apis/LoginFlow/index.html#login-flow-v2). It prints a link: open it in a browser, sign in and grant access. Nextcloud then creates an app password named "Tickcli on *hostname*".

- `--paste`: type a user and an app password you created yourself, instead of opening a link. Use an **app password**, never your account password. When standard input isn't a terminal, the user and the app password are read as two lines from it.
- `--qr`: also show the link as a QR code, to sign in from a phone.
- `--insecure-http`: allow plain `http://` to a public address.

Tickcli stores the credentials in the system keychain when there is one, otherwise in an encrypted file (see FILES), and says which it used. Only one account is signed in at a time.

### logout

Forget the stored credentials. If Tickcli created the app password through `login`, it also revokes it on the server. An app password you pasted is forgotten but not revoked: remove it in Nextcloud under Settings → Security → Devices & sessions if nothing else uses it.

- `--local`: only forget the credentials, without revoking the app password.

`logout` doesn't touch the `TICKBUDDY_*` environment variables. When they are set, it still forgets what is stored and reminds you that Tickcli keeps using them.

### status

Show who is signed in, at which server, the Tickbuddy version, and where the credentials come from. If the server lacks features Tickcli uses, `status` lists them by their feature flag, such as `numberTracks`, and says what that affects.

```
$ tickcli status
Signed in as martin at https://cloud.example.com · Tickbuddy 3.2.0 · credentials in keychain
```

### tick *track*, untick *track*

Tick or untick a Yes/No track on a day (default: today). Both take the note options.

### set *track* *value*

Set a track's value on a day (default: today). Takes the note options.

- **Counter:** a whole number. `0` removes the count.
- **Number:** a number such as `10.4` or `-1.5`. The server rounds it to the track's decimal places.
- **Choice:** an option's label, ignoring case, or `'#2'` for the second option.

`""` clears a number or choice day.

### add *track* [*n*]

Add *n* (default 1) to a counter. A negative *n* takes away, never going below 0: `tickcli add coffee -2`. Takes the note options.

`add` is for counters only. To add to a number track, read the day with `tickcli ticks` and set the new total with `tickcli set`. `add` is never retried (see Retries).

### note *track* *text*

Set or replace a day's note, on a track of any type.

- `--append`: add the text to the note on a new line instead, unless the note already holds it.
- `--force-duplicate`: with `--append`, append the text even if the note already holds it.
- `--clear`: delete the note, instead of giving a text.

### tracks

List tracks with their ids and types, each number track's unit and decimal places, and each choice track's options. `--archived` and `--private` add those tracks.

### ticks [*track*]

Show a track's value on a day (default: today), or on every day of a range:

- `--from date`: the first day of the range. The range runs to today, or to `--to`.
- `--to date`: the last day of the range. Needs `--from`.
- `--year YYYY`: 1 January to 31 December of that year, up to today in the current year.

`-d` and a range can't be combined. Every day in the range is listed: an unticked day as `no` or `0`, and a number or choice day without a value as `·` (an empty field when piped).

Without a track, `ticks` shows the tracks of the today view for each day. `--private` and `--archived` add those tracks.

### week

Show the last 7 days of every track, today first, with the current streak or break. At a terminal, each day's column is 2 characters wide, or wide enough for the longest number when a number or choice track is shown. Longer choice labels are cut. Piped and with `--json`, every value is in full.

### notes

List the notes of the last 7 days, or of a range:

- `--from date`: the first day. Default: 6 days before `--to`.
- `--to date`: the last day. Default: today.

### stats [*track*]

Show the statistics Tickbuddy shows on Analytics: the total, the weekly mean, the last 7 days against the previous 7, the current streak or break, and the longest streak and break. Without a track, it shows them for every track.

By default the statistics cover all time. To cover a period, use `--from` and `--to` (either alone), or `--year`, but not both:

- `--from date`: the first day of the period. Default: the first tick. `--from -30` is the last 30 days and today.
- `--to date`: the last day of the period. Default: today.
- `--year YYYY`: 1 January to 31 December of that year.

For a period, "Last 7 days" and "Previous 7" are the period's last two weeks. A period that ended before today shows its streak or break as of its last day, labelled "At the end". A period needs a server with period statistics (feature flag `statsPeriod`).

On number and choice tracks, every count is a count of days with an entry.

### export

Write the server's JSON export to standard output, unchanged. It is the same file as an export from Tickbuddy's settings, so it works as a backup: `tickcli export > backup.json`. `--private` includes private tracks.

### help [*command*]

Show help for Tickcli or for one command. `tickcli command --help` does the same.

## OPTIONS

These options work with every command, wherever they apply.

- `-d` *date*, `--date` *date*: the day to read or write. Default: today. See Dates.
- `--json`: write JSON instead of text. See OUTPUT.
- `--numeric`: show a choice by its place instead of its label (`2` for Happy), and Yes/No days as `1` and `0`. Applies to the today view, `ticks`, `week` and `set`. `--json` output doesn't change.
- `--private`: include private tracks in lists.
- `--archived`: include archived tracks in lists.
- `--retries` *n*: how many times to retry a safe request when the server can't be reached. Default: 3.
- `-h`, `--help`: show help.
- `-v`, `--version`: show Tickcli's version.

Options can come before or after the arguments: `tickcli tick exercise --json`. A bare negative number is read as a value, not an option, so `tickcli add coffee -2` and `tickcli set "Greenhouse low" -1.5` work.

## OUTPUT

Tickcli writes in one of three ways.

- **At a terminal:** formatted, in colour. Set `NO_COLOR` (any value) or `TERM=dumb` to turn colour off.
- **Piped:** tab-separated lines, with dates as `YYYY-MM-DD`. A write prints nothing when it succeeds, so a script's log stays quiet.
- **With `--json`:** JSON, on every command.

Errors always go to standard error, as `tickcli: message`, or with `--json` as `{"error": "message", "code": n}`, where `n` is the exit status.

### Piped `ticks`

Each line is the date, the track, the value, and the note if there is one, oldest day first. The value is:

- `yes` or `no` on a Yes/No track (`1` or `0` with `--numeric`)
- the count on a counter
- the number on a number track, at the track's decimal places and without the unit (`10.4`)
- the option's label on a choice track (its place with `--numeric`)
- an empty field on a number or choice day without a value

### JSON from `ticks`

```json
{"from": "2026-11-17", "to": "2026-11-23",
 "tracks": [{"track": "Run", "trackId": 4, "type": "number",
             "days": [{"date": "2026-11-23", "value": 10.4, "unit": "km", "note": "Easy pace"}]}]}
```

Every day of the range is listed, even for a single day. `note` appears only when there is one. A number day adds `unit` and gives the value exactly as stored; a choice day gives the option's place as `value` and adds its `label`.

### JSON from a write

`tick`, `untick`, `set` and `add` answer with `{"track", "trackId", "date", "value"}`. The `value` is:

- `true` or `false` on a Yes/No track
- the count on a counter
- the stored number on a number track, with `unit` when the track has one
- the option's place on a choice track, with its `label`
- `null` after clearing a day

When the command set or appended a note, `note` holds the whole note afterwards.

## ENVIRONMENT

`TICKBUDDY_URL`, `TICKBUDDY_USER`, `TICKBUDDY_APP_PASSWORD`
: The server, the user and an app password. When all three are set, Tickcli uses them instead of stored credentials, and stores nothing. Use them for containers and CI, or to reach another account for one run.

`NO_COLOR`
: Any value turns colour off.

`TERM`
: `dumb` turns colour off.

`XDG_CONFIG_HOME`
: Where the credentials file is kept, when Tickcli uses one. Default: `~/.config`.

## FILES

Tickcli looks for credentials in this order:

1. The `TICKBUDDY_*` environment variables.
2. The system keychain: Keychain on macOS, Credential Manager on Windows, and the Secret Service (GNOME Keyring, KWallet) on Linux desktops.
3. The file `~/.config/tickcli/credentials` (or `$XDG_CONFIG_HOME/tickcli/credentials`), on Linux and the BSDs when there is no keychain, such as on a headless server.

The file is encrypted. Where per-user `systemd-creds` works (systemd 256 or newer, such as Debian 13 and Ubuntu 24.10 and later), it is encrypted with that, bound to the TPM if there is one and otherwise to a host key only root can read. Elsewhere it is encrypted with a key derived from `/etc/machine-id` and your user id.

The encryption means a copy of the file that leaks, through a backup or a dotfiles repository for example, can't be read on another machine. It does **not** protect against someone who can log in as you on the same machine: they can run Tickcli as you, or derive the same key.

TickLLMe reads the same credentials.

## EXIT STATUS

| Status | Meaning |
|---|---|
| 0 | Success. |
| 1 | Error, including a server too old for the command. |
| 2 | Bad usage: an unknown command or option, wrong arguments, a bad date, the wrong command for the track's type, a value that isn't a number on a number track, an option the choice track doesn't have, or a track of a type this version of Tickcli doesn't know. Nothing was sent. |
| 3 | Not signed in, or the server refused the credentials. |
| 4 | The track wasn't found, or the name matches several tracks. |
| 5 | The server or the network can't be reached. |
| 6 | The track is read-only (archived), so nothing was changed. |

## EXAMPLES

Tick a track, and add to a counter with a note:

```
$ tickcli tick exercise
✓ Exercise · Mon 23 Nov

$ tickcli add coffee 2 --note "one decaf"
Cups of coffee: 5 · Mon 23 Nov  📝 one decaf
```

Set a number and a choice:

```
$ tickcli set run 10.4
Run: 10.4 km · Mon 23 Nov

$ tickcli set "mum's mood" happy
Mum's mood: Happy · Mon 23 Nov
```

Log a nightly backup from cron, with the time it finished as the note:

```
30 2 * * * restic backup /srv && tickcli tick "Nightly backup" --note "$(date +%H:%M)"
```

Log a sensor reading every morning. An empty reading clears the day:

```
0 7 * * * tickcli set "Greenhouse low" "$(read-sensor)"
```

Add each activity's link to the day's note. Running it again for the same activity changes nothing:

```
tickcli tick Exercise --append-note "https://www.strava.com/activities/…"
```

Read values in a script:

```
$ tickcli ticks "Nightly backup" -d yesterday | cut -f3
yes

$ tickcli ticks Exercise --year 2025 --json | jq '[.tracks[].days[] | select(.value)] | length'
212
```

Compare a track year on year:

```
tickcli stats Exercise --year 2025
tickcli stats Exercise --year 2026
```

Handle failures in a script by exit status:

```
tickcli tick "Nightly backup" --note "$(date +%H:%M)"
case $? in
  0) ;;
  5) echo "Tickbuddy unreachable; try again later" >&2 ;;
  6) echo "Nightly backup track is read-only; nothing was recorded" >&2 ;;
  *) echo "tickcli failed" >&2 ;;
esac
```

## SEE ALSO

- [TickLLMe](@/docs/tickllme.md), which uses Tickcli's sign-in
- [API reference](@/docs/api.md)
- [Tickcli on GitHub](https://github.com/martinhammer/tickcli)
