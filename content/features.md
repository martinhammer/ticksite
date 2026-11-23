+++
title = "Features"
+++

## Concepts
The key Tickbuddy concept is the **track**. Tracks represent the habit, event, or occurrence to be tracked daily (_Exercise_, _Cups of coffee_, _CI pipeline failure_). The track value on a particular day is called a **tick** (_Exercise on 2 Jan: Yes_, _Cups of coffee on 24 Oct: 2_).

Each track has a **type**, and as of today there are four types: Yes/No, Counter, Number and Choice. Further track types are under consideration.

Optionally, it is possible to attach a short **note** to each tick (e.g. _Run cut short: injury flared up_, _Backup completed successfully at 23:51_).

Tracks can optionally be marked as **private**, which means that clients do not show them by default unless requested. The home screen and dashboard widgets never display private tracks. This feature supports logging of sensitive data.

Tracks can be **archived**, which means that they become read-only and clients do not display them, but the full history is kept. Archived tracks can be restored.

## Existing functionality

Key features that are available now:

### Tickbuddy (backend)
* stores tracks, ticks, notes
* four track types Yes/No, Counter, Number, Choice
* tracks can be marked as private
* tracks can be archived and restored, renamed, deleted, reordered
* Number and Choice types have options that users can set  
* everything is exposed through [APIs](@/docs/api.md) which all of the clients leverage
* public links / ping URLs for script and automation integrations without an app password

### Tickbuddy (web UI)
* track management
* Today and Journal screens to view and edit your daily log
* per track Analytics screen
* dashboard widget
* export and import functionality

### Tickdroid
* connect to Tickbuddy with an app password
* Today and Journal screens to view and edit your daily log
* full track management including add, rename, reorder, toggle private, archive/restore, delete
* interactive home screen widgets
* app settings including light and dark mode, journal grid density, etc.

### Tickcli ("tickly")
* connect to Tickbuddy with Login Flow v2 or an app password
* comprehensive set of read and write commands
* pretty-print output to terminal, plain output when piped, optional JSON output
* refer to [documentation](@/docs/tickcli.md)

### TickLLMe ("tickle me")
* a ready-made extension (.mcpb) for Claude Desktop, and a plugin for Claude Code
* connect to Tickbuddy with an app password or piggyback on an existing Tickcli login
* three permission levels (read only; also record ticks; also add and rename tracks)
* refer to [documentation](@/docs/tickllme.md)

## Roadmap
These are some of the planned features (not in the order of priority):

* Setting and tracking of targets, goals and expectations
* Defining groups of tracks
* Stats and analytics in Tickdroid
* Improved analytics in Tickbuddy
* Remote MCP server
* TickRED: Tickbuddy node for Node-RED
* Migration options from other tools
* Support for translations and improved accessibility

Some early stage ideas, which are not yet committed as part of the roadmap:

* Shared tracks with other users on the same Nextcloud instance
* Immutable / read-only tracks, i.e. value cannot be changed once set or beyond the end of the day
* Further track types such as duration and time of day
* Calculated / derived tracks, i.e. value is calculated from other tracks
* Composite tracks, i.e. tracks that have more than one value
* Garmin smartwatch app

Feel free to [get in touch](@/contact.md) if you have another request, suggestion or idea.
