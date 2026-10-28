+++
title = "Features"
+++

## Concepts
The key Tickbuddy concept is the **track**. Tracks represent the habit, event, or occurrence to be tracked daily (_Exercise_, _Cups of coffee_, _CI pipeline failure_). The track value on a particular day is called a **tick** (_Exercise on 2 Jan: Yes_, _Cups of coffee on 24 Oct: 2_).

Each track has a **type**, and as of today there are two types: Yes/No and Counter. More track types are planned.

Some clients (specifically Tickcli and TickLLMe; web and Android will follow) already support attaching a short **note** to each tick (e.g. _Run cut short: injury flared up_, _Backup completed successfully at 23:51_).

Tracks can optionally be marked as **private**, which means that clients should not show them by default unless requested.

Tracks can be **archived**, which means that they become read-only and clients should not show them, but the full history is kept. Archived tracks can be restored.

## Existing functionality

Key features available now:
* Tickbuddy (backend)
	* stores tracks, ticks, notes
	* track types Yes/No and Counter
	* tracks can be marked as private
	* tracks can be archived and restored, renamed, deleted, reordered
	* everything is exposed through APIs which all of the clients leverage
* Tickbuddy (web UI)
	* track management
	* Edit Journal and View Journal screens
	* per track Analytics screen
	* dashboard widget
	* export and import functionality
* Tickdroid
	* connect to Tickbuddy with an app password
	* journal screen to view and tick
	* full track management including add, rename, reorder, toggle private, archive/restore, delete
	* interactive home screen widgets
	* app settings including light and dark mode, journal grid density, etc.
* Tickcli ("tickly")
	* connect to Tickbuddy with Login Flow v2 or an app password
	* comprehensive set of read and write commands (including notes, not yet available on the web or Android)
	* pretty-print output to terminal, plain output when piped, optional JSON output
* TickLLMe ("tickle me")
	* a ready-made extension (.mcpb) for Claude Desktop, and a plugin for Claude Code
	* connect to Tickbuddy with an app password or piggyback on an existing Tickcli login
	* three permission levels (read only; also record ticks; also add and rename tracks)

## Roadmap

These are some of the major planned features. Feel free to [get in touch](@/contact.md) if you have another request.
* UI updates for web and Android (coming soon!)
* Attaching short notes to ticks in Tickbuddy and Tickdroid (coming soon!)
* Ping URLs for further automation integrations, e.g. home automation and TickRED for Node-RED
* Additional track types (numbers and custom lists)
* Setting and tracking of targets, goals and expectations
* Defining groups of tracks
* Improved analytics
* Migration options from other tools
* Support for translations and improved accessibility
* Maybe even a smartwatch app
