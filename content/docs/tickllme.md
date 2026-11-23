+++
title = "TickLLMe"
description = "MCP server that lets AI agents interact with Tickbuddy"
weight = 3
+++

This page describes TickLLMe 0.2.0, released 22 November 2026.

## NAME

`tickllme` - the MCP server that lets your AI agents log and read your entries in Tickbuddy, the daily habit and event tracker for Nextcloud. Pronounced "tickle me".

## SYNOPSIS

```
tickllme [serve] [--level 1|2|3] [--private]
tickllme login <url> [--paste] [--insecure-http]
tickllme logout [--local]
tickllme status
tickllme help
tickllme --version
```

## DESCRIPTION

TickLLMe lets Claude, or another AI assistant that supports MCP (the Model Context Protocol), answer questions about your Tickbuddy journal: "how's my exercise streak?", "what did I tick yesterday?". If you allow it, it can also record things ("I had three coffees", "tick the nightly backup and note it finished at 02:14") and create tracks.

TickLLMe runs on your own computer and talks only to your Nextcloud. Your assistant starts it and talks to it over standard input and output, so you rarely run it yourself. You choose what the assistant may do with an access level (see LEVELS), and private tracks stay hidden unless you turn them on.

Your assistant can never delete, archive or reorder tracks, and it can't import anything. Claude also asks before each tool call unless you allow that tool.

### Tracks

Tickbuddy has four types of track: Yes/No, counter, number and choice. Each track can have one plain-text note per day, of up to 500 characters.

A tool names a track by its exact name, ignoring case, or by its id as `#n`. Unlike Tickcli at a terminal, a prefix never matches. If a name doesn't match, the error lists the tracks, so the assistant can ask you which one you meant.

### Dates

Dates are `YYYY-MM-DD`, `today`, `yesterday` or `-N` for N days ago, in your computer's time zone. Every tool result includes `today`, so the assistant uses your date rather than its own idea of it.

### Numbers and choices

- A number uses a point as the decimal separator and no grouping: `10.4`, not `10,4` or `1,204.5`.
- Tickbuddy rounds a number to the track's decimal places, and the assistant reports the value it stored.
- A choice is set by its option's label, `"#2"` for the option in place 2, or the place as a number.
- `null` clears a number or choice day. On those tracks, a number's 0 is a real reading.
- Only counters can be added to. To add a second run to today's, the assistant reads the day and sets the new total.

### Notes

Writing a note replaces the day's note. Appending adds the text on a new line instead, and a line the note already holds is skipped, so an assistant logging several activities a day can append safely.

## INSTALLATION

### Claude Desktop

Download `tickllme-v0.2.0.mcpb` from the [latest release](https://github.com/martinhammer/tickllme/releases/latest) and add it under Extensions in Claude Desktop. Then sign in and choose the access level in the extension's settings (see SETTINGS). It runs on macOS, Windows and Linux.

### Claude Code

```
/plugin marketplace add martinhammer/tickllme
/plugin install tickllme@tickllme
```

Claude Code then asks for the settings. It needs Claude Code 2.1.224 or later, on Linux (x64 or arm64), macOS or Windows.

Claude Code doesn't update plugins from marketplaces other than Anthropic's by default. Turn on auto-update for this marketplace in `/plugin`, or update by hand there.

### Other MCP clients

Any MCP client that runs local servers over standard input and output can run `tickllme serve`. Download the plugin zip from the [latest release](https://github.com/martinhammer/tickllme/releases/latest), which holds builds for Linux, macOS and Windows. Point your client at the binary for your system, with the arguments `serve --level N`, and sign in with `tickllme login` or the `TICKBUDDY_*` variables (see ENVIRONMENT).

## SETTINGS

Claude Desktop and Claude Code show these settings for TickLLMe.

- **Nextcloud address:** the address you open Nextcloud at, such as `https://cloud.example.com`. Only `https://` is accepted, or plain `http://` to `localhost` or a private IP address. For a Nextcloud you reach over plain http by name, such as `http://nextcloud.local`, sign in with `tickllme login url --insecure-http` instead and leave the three sign-in fields empty.
- **Nextcloud user name:** your Nextcloud login name.
- **Nextcloud app password:** an app password, never your normal password. Create one in Nextcloud under Settings → Security → Devices & sessions ("Create new app password"). Use one just for TickLLMe, so you can revoke it on its own.
- **What Claude may do:** the access level, 1 to 3. Default: 1. See LEVELS.
- **Show private tracks:** off, the assistant can't see or change tracks you've marked private; on, it can. Default: off.

### Signing in

There are two ways, and you only need one:

- Fill in all three sign-in fields: Nextcloud address, user name and app password.
- Leave all three empty if you've already run `tickcli login` or `tickllme login` on this computer. TickLLMe then uses that sign-in, from the keychain or Tickcli's encrypted file.

If both are present, the settings win. `tickllme status` shows which sign-in is used.

## LEVELS

The access level decides which tools your assistant sees. Tools above the level aren't offered at all.

| Level | Your assistant can |
|---|---|
| 1 | Only read your journal: tracks, days, ticks, notes and stats. The default. |
| 2 | Also tick and untick, change counters, log numbers and choices, and write notes. |
| 3 | Also create and rename tracks, and rename a choice track's options. |

## TOOLS

Each tool answers with JSON that includes `today`. An error comes back as one plain sentence the assistant can pass on to you, such as `Track "Nightly backup" is archived, so nothing was changed`.

### Level 1: reading

`list_tracks`
- The tracks in journal order: id, name, type and whether archived. A number track adds its unit, decimal places and whether it allows negative values. A choice track adds its options in order, each with its place and label, and whether they are an ordered scale. A track of a type newer than TickLLMe shows that type's name, and can be read but not set.
- `include_archived`: include archived tracks.

`get_day`
- Every track on one day, with its value and note. Unticked tracks are included, as `false` or `0`.
- `date`: the day. Default: today.
- `include_archived`: include archived tracks.

`get_ticks`
- The entries from one day to another, both included. Only days with an entry are listed. At most 31 days across all tracks, or 366 days for one track, and 500 entries.
- `from`, `to`: the first and last day. Required.
- `track`: only this track.

`get_notes`
- The day notes from one day to another, both included, newest first. The same limits as `get_ticks`.
- `from`, `to`: the first and last day. Required.
- `track`: only this track.

`get_stats`
- The statistics Tickbuddy shows on Analytics, per track: the total, the weekly mean, the last 7 days, and the current and longest streak and break. On number and choice tracks, the counts are of days with an entry. `maxValue` and `minValue` are the highest and lowest number on a number track, or place on a choice track with an ordered scale, and `null` on a choice track with a plain list.
- `track`: only this track. Default: every track.
- `include_runs`: also list every streak and break. Needs `track`.
- `from`, `to`: count only this period. Default: from the first tick to today. Comparing two years is two calls. A period needs a server with period statistics (feature flag `statsPeriod`).

### Level 2: writing

`set_tick`
- Sets one day's value on any type of track, optionally with a note. Safe to repeat.
- `track`: the track. Required.
- `value`: required. On a Yes/No track, `true` or `false`. On a counter, a whole number, 0 or more; 0 removes the entry. On a number track, a number such as `10.4` or `"10.4"`. On a choice track, an option's label, `"#N"` for the option in place N, or N as a number. `null` clears a number or choice day.
- `date`: the day. Default: today.
- `note`: replace the day's note with this. `""` deletes it. Leave it out to keep the note.
- `append_note`: add this to the day's note on a new line. A line already there is skipped. Not with `note`.

`add_to_counter`
- Adds to a counter on one day, or takes away with a negative number, never going below 0. Optionally with a note. **Not safe to repeat**: each call counts again.
- `track`: the counter. Required.
- `delta`: how much to add, not 0. Required.
- `date`, `note`, `append_note`: as for `set_tick`.

`set_note`
- Sets, replaces or deletes the note of a track on one day, or adds to it.
- `track`: the track. Required.
- `text`: the note, plain text, up to 500 characters. `""` deletes the note, unless `append` is set. Required.
- `date`: the day. Default: today.
- `append`: add the text on a new line instead of replacing the note. A line already there is skipped. Needs a server with appending (feature flag `notesAppend`).

### Level 3: tracks

`create_track`
- Adds a new track at the end of the journal.
- `name`: the new track's name. Required.
- `type`: `yes/no`, `counter`, `number` or `choice`. Required.
- `unit`: number only. The unit, such as `km` or `°C`. Default: none.
- `decimal_places`: number only. How many decimals to keep, 0 to 3. Default: 1.
- `allow_negative`: number only. Whether values below 0 are allowed. Default: false.
- `options`: choice only. 2 to 99 option labels, in order. They can't be added, removed or reordered later, only renamed.
- `ordered_scale`: choice only. Whether the options run from one end of a scale to the other, such as Excited to Sad, rather than being a plain list. Default: false.

`rename_track`
- Gives a track a new name. Its entries and notes stay.
- `track`: the track. Required.
- `name`: the new name. Required.

`rename_option`
- Gives one option of a choice track a new label. The days that hold it keep it.
- `track`: the choice track. Required.
- `option`: the option, by its current label, `"#N"` for the option in place N, or N as a number. Required.
- `label`: the new label. Required.

A number track's unit, decimal places and negative values, and a choice track's ordered scale, can only be changed in Tickbuddy on the web.

### Prompts

`log_day` ("Log my day") goes through every track for a day, confirms what's already entered and asks about the rest, then saves the changes after one yes. It takes `date`, default today, and needs level 2. In Claude Code it is `/mcp__tickbuddy__log_day`; Claude Desktop offers it among the extension's prompts.

## COMMANDS

### serve

Serve MCP over standard input and output. This is what your assistant runs, and it is the default when no command is given. Standard output carries only MCP messages; logs go to standard error.

- `--level 1|2|3`: the access level. Default: 1.
- `--private`: show private tracks. `--private=false` hides them, the default.

### login *url*

Sign in to a Nextcloud that has Tickbuddy, as `tickcli login` does. By default, TickLLMe prints a link: open it in a browser, sign in and grant access. Nextcloud then creates an app password named "TickLLMe on *hostname*". The sign-in is shared with Tickcli, so leave the sign-in settings empty to use it.

- `--paste`: type a user and an app password you created yourself, instead of opening a link. When standard input isn't a terminal, they are read as two lines from it.
- `--insecure-http`: allow plain `http://` to a public address, or to a name such as `nextcloud.local`.

### logout

Forget the stored sign-in, for both Tickcli and TickLLMe. If the app password was created through `login`, it is also revoked on the server. An app password you pasted is forgotten but not revoked.

- `--local`: only forget the sign-in, without revoking the app password.

Sign-in settings filled in for TickLLMe are not affected.

### status

Show who is signed in, at which server, where the sign-in comes from, and the Tickbuddy version, with any features the server lacks.

```
$ tickllme status
Signed in as martin at https://cloud.example.com · from TickLLMe's settings
Tickbuddy 3.2.0
```

### help, --version

Show the usage, or TickLLMe's version.

## ENVIRONMENT

- `TICKBUDDY_URL`, `TICKBUDDY_USER`, `TICKBUDDY_APP_PASSWORD`: the server, the user and an app password. When all three are set, TickLLMe uses them instead of the stored sign-in. The plugin and the extension pass TickLLMe's sign-in settings this way.
- `TICKLLME_LEVEL`: the access level, as `--level`. The flag wins.
- `TICKLLME_PRIVATE`: `true` shows private tracks, as `--private`. The flag wins.

## FILES

TickLLMe shares its stored sign-in with Tickcli: the system keychain, or the encrypted file `~/.config/tickcli/credentials` where there is no keychain. See FILES in [Tickcli](@/docs/tickcli.md) for what the file protects.

## EXIT STATUS

| Status | Meaning |
|---|---|
| 0 | Success. |
| 1 | Error. |
| 2 | Bad usage: an unknown command or option, or a level other than 1, 2 or 3. |

## EXAMPLES

Use TickLLMe from another MCP client, signed in through Tickcli, with writing allowed:

```json
{
  "mcpServers": {
    "tickbuddy": {
      "command": "/path/to/tickllme",
      "args": ["serve", "--level", "2"]
    }
  }
}
```

Check which sign-in TickLLMe will use:

```
tickllme status
```

Things to ask your assistant:

- "How's my exercise streak?"
- "What did I tick yesterday?"
- "Compare my running in 2025 and 2026."
- "I had three coffees." (level 2)
- "Tick the nightly backup and note it finished at 02:14." (level 2)
- "Create a track called Coding and tick the days in 2026 I made a contribution on GitHub." (level 3)

## SEE ALSO

- [Tickcli](@/docs/tickcli.md), whose sign-in TickLLMe shares
- [API reference](@/docs/api.md)
- [TickLLMe on GitHub](https://github.com/martinhammer/tickllme)
- [Model Context Protocol](https://modelcontextprotocol.io)
