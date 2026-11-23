+++
title = "API reference"
description = "The REST API that all client tools use"
weight = 1
+++

This page describes the API of Tickbuddy 3.2.0, released 22 November 2026. It is the API that the web app, Tickdroid, Tickcli and TickLLMe use. 

## Conventions

### Base URL

```
{nextcloud}/ocs/v2.php/apps/tickbuddy
```

All paths on this page are relative to it, except capabilities, which is a core Nextcloud endpoint. The Nextcloud address is the user's own server.

### Headers

Send these on every request:

| Header | Value | Why |
|---|---|---|
| `OCS-APIRequest` | `true` | Required by Nextcloud. Without it you get an HTML page or a 404. |
| `Accept` | `application/json` | Returns the JSON envelope instead of XML. |
| `Authorization` | `Basic base64(login:app-password)` | A Nextcloud app password, never the user's login password. No CSRF token is needed. |

These are optional:

| Header | Value | Since | Why |
|---|---|---|---|
| `Prefer` | `handling=strict` | 2.3.0 | [RFC 7240](https://www.rfc-editor.org/rfc/rfc7240). A change to an archived track answers 423 instead of a quiet 200 that changes nothing (see Archived tracks). Recommended on every request. The server confirms with `Preference-Applied: handling=strict`; without that in the answer, the old behaviour applies. |
| `Idempotency-Key` | `"<uuid>"` | 2.3.0 | On the writes that aren't safe to repeat, so a retry is answered instead of done twice. See Retrying writes. |

Send `POST` and `PUT` bodies form-encoded or as JSON; Nextcloud reads both.

### Booleans

Since 3.1.0, every boolean parameter takes `true`/`false`, `1`/`0`, `on`/`off` or `yes`/`no`, in any letter case, or a JSON boolean or the numbers 1 and 0. Anything else, an empty value included, answers 400 `<name> must be true or false` (for example `includeArchived must be true or false`). That check comes before anything else, and nothing is changed.

Servers before 3.1.0 read `no` and `off` as true, so send `true` or `false`.

### Response envelope

```json
{ "ocs": {
    "meta": { "status": "ok", "statuscode": 200, "message": "OK" },
    "data": … } }
```

Unwrap `ocs.data`. Errors use the same envelope with `status: "failure"` and `data: {"message": "…", "reason": "…"}` (`reason` since 2.3.0, see Errors).

Use `/ocs/v2.php`. On `/ocs/v1.php`, every answer arrives as HTTP 200, with the real status only in `statuscode`.

### Status codes

| Code | Meaning |
|---|---|
| 200 | OK: reads and most writes. |
| 201 | Created: a track or a public link. |
| 204 | Done, no body: deleting a track or a note, revoking a public link. |
| 400 | Bad input, or the wrong endpoint for the track's type. |
| 401 | Credentials missing or revoked. |
| 403 | A limit reached: 99 tracks, or 99 public links on a track (since 3.1.0). A day outside a public link's allowed days (since 3.1.0). |
| 404 | Track not found, or not the user's. Public link not found or revoked (since 3.1.0). Option not found (since 3.2.0). |
| 409 | A conflict: see Errors. |
| 422 | An `Idempotency-Key` already used for a different request (since 2.3.0). |
| 423 | The track is archived (since 2.3.0). See Archived tracks. |
| 429 | Too many imports or public link calls (since 3.1.0). Nextcloud's own answer, with a body that isn't OCS JSON. |

### Data rules

- **Dates** are `YYYY-MM-DD` in the user's local calendar. The server does no time-zone maths: the day is whatever the client says.
- **Sparse storage.** A tick row means ticked, or a counter above 0. No row means not ticked, or 0. A Yes/No tick always has the value 1. A Number or Choice row is a day with an entry, a Number's 0 included; no row means no entry.
- **Track types** are `boolean` (Yes/No), `counter`, `number` (a reading with a unit and 0 to 3 decimal places, since 3.2.0) and `choice` (one of the track's options, since 3.2.0). A track's type is fixed when it is created. Reads leave out Number and Choice tracks unless the client sends `includeAllTypes=true`.
- **`private` is a display hint**, not access control. Every endpoint returns private tracks and their ticks, and clients hide them.
- **No pagination.** A user has at most 99 tracks, archived ones included. Ticks are bounded by the date range asked for.

## Capabilities

### GET /ocs/v2.php/cloud/capabilities

*Since 1.0.6.* A core Nextcloud endpoint, so not under the Tickbuddy base URL; send the same headers. Read `data.capabilities.tickbuddy` once after signing in. If it is missing, either Tickbuddy isn't installed or the server is older than 1.0.6: `GET /api/tracks` tells the two apart. Branch on the feature flags or `apiVersion`, never on the version string.

```json
{ "version": "3.2.0",
  "apiVersion": 1,
  "features": { "import": true, "export": true, "counterTracks": true,
                "privateTracks": true, "tickBounds": true, "archivedTracks": true,
                "syncDelta": false, "counterIncrement": true,
                "booleanSet": true, "stats": true, "statsPeriod": true,
                "notes": true, "notesAppend": true, "idempotencyKey": true,
                "trackFilter": true, "notesBounds": true, "publicLinks": true,
                "numberTracks": true, "choiceTracks": true } }
```

`apiVersion` has been 1 throughout, because every change so far has only added things.

| Flag | Since | Means |
|---|---|---|
| `import`, `export` | 1.0.6 | The backup endpoints exist (they have since 1.0.0). |
| `counterTracks`, `privateTracks` | 1.0.6 | Counter tracks and the private flag exist (they have since 1.0.0). |
| `syncDelta` | 1.0.6 | Reserved, always `false`: there is no "changed since" endpoint yet. |
| `tickBounds` | 1.1.0 | `GET /api/ticks/bounds` exists. Without it, the route answers 404. |
| `archivedTracks` | 2.2.0 | Archiving and restoring, `archivedAt` and `includeArchived`. |
| `counterIncrement` | 2.3.0 | `POST /api/ticks/increment` exists. Reserved and `false` before 2.3.0. |
| `booleanSet` | 2.3.0 | `POST /api/ticks/set` takes Yes/No tracks. |
| `stats` | 2.3.0 | `GET /api/stats` exists. |
| `statsPeriod` | 2.3.0 | `from` and `to` on stats. Check it before sending them: an older server ignores them and answers all-time numbers. |
| `notes` | 2.3.0 | The notes endpoints, and `note` on tick calls. An older server ignores `note` without an error. |
| `notesAppend` | 2.3.0 | `POST /api/notes/append`, `appendNote`, and the 500-character limit (200 on a server with `notes` but without this flag). |
| `idempotencyKey` | 2.3.0 | The writes that aren't safe to repeat honour `Idempotency-Key`. An older server ignores the header. |
| `trackFilter` | 3.1.0 | `GET /api/ticks` and `GET /api/notes` take `trackId`. An older server ignores it and answers every track, so without this flag never assume the answer holds one track. |
| `notesBounds` | 3.1.0 | `GET /api/notes/bounds` exists. Without it, the route answers 404. |
| `publicLinks` | 3.1.0 | `/api/public-links`, `publicLinkCount` on tracks, and the call at `/ping/{token}`. Without it, the routes answer 404. |
| `numberTracks` | 3.2.0 | Number tracks, their fields, `set` on them, and `includeAllTypes`. An older server refuses the type when creating a track (400) and ignores `includeAllTypes`. |
| `choiceTracks` | 3.2.0 | Choice tracks, `options` and `orderedScale`, `set` on them, `includeAllTypes`, and renaming an option, whose route older servers answer with 404. |

## Errors

*Since 2.3.0*, every error answer from Tickbuddy carries a machine-readable `reason` next to `message`. Show `message` to people and never parse it; branch on `reason`.

```json
{"message": "Track not found", "reason": "trackNotFound"}
```

| Status | `reason` | Since | When |
|---|---|---|---|
| 400 | `invalidValue` | 2.3.0 | A parameter is missing or wrong, including an `Idempotency-Key` that isn't a UUID. `message` says which. |
| 400 | `wrongTrackType` | 2.3.0 | The endpoint doesn't take this track's type: `toggle` on a counter, `increment` on a Yes/No track, either on a Number or Choice track, or renaming an option of a track that isn't a Choice. |
| 400 | `invalidUpload` | 2.3.0 | An import with no file, an upload error, or a file over 20 MB. |
| 400 | `importFailed` | 2.3.0 | The import file was read and refused. Nothing was imported. |
| 403 | `trackLimitReached` | 2.3.0 | The user has 99 tracks already. |
| 403 | `publicLinkLimitReached` | 3.1.0 | The track has 99 public links already. |
| 403 | `dayNotAllowed` | 3.1.0 | A public link call names a day outside the link's allowed days. Nothing was done. |
| 404 | `trackNotFound` | 2.3.0 | The track doesn't exist or isn't the user's. |
| 404 | `publicLinkNotFound` | 3.1.0 | The public link doesn't exist, isn't the user's or is revoked. On a call, also when its owner can't sign in. |
| 404 | `optionNotFound` | 3.2.0 | The Choice track has no option at this place. |
| 409 | `noteConflict` | 2.3.0 | Other writers kept changing the note while appending. |
| 409 | `databaseConflict` | 2.3.0 | Another change took a track's place in the list meanwhile: `The tracks changed while saving; try again` on `PUT /api/tracks/{id}`, or `The tracks changed while reordering; try again` on reorder. Nothing changed; read the tracks again and retry. |
| 409 | `requestInProgress` | 2.3.0 | A request with the same `Idempotency-Key` is still being handled. Try again shortly. |
| 422 | `idempotencyKeyReused` | 2.3.0 | The `Idempotency-Key` was already used for a different request. Nothing was written. |
| 423 | `archived` | 2.3.0 | The track is archived. See Archived tracks. |

- **Accept unknown values.** Treat a `reason` you don't know like an error with only a message. New values come without a feature flag, and values are never renamed.
- **No `reason` means the answer isn't Tickbuddy's.** Nextcloud answers some errors itself: a disabled app's 404 (during an upgrade, for example), 503 in maintenance mode, 401, a bare 400 for a parameter it can't convert, a 500, and the 429 on imports and public link calls. Treat those as temporary, and don't drop a queued write on a 404 or 400 without a `reason`. A server before 2.3.0 sends no `reason` at all.

### Archived tracks

*Since 2.3.0.* With `Prefer: handling=strict`, every change to an archived track except deleting and restoring it answers `423 Locked` and changes nothing. That covers `toggle`, `set`, `increment`, `notes/set`, `notes/append`, `DELETE /api/notes`, `PUT /api/tracks/{id}` with `name`, `private`, `sortOrder` or `archived=true`, and a `reorder` that lists the track.

```json
{"message": "Exercise is archived, so nothing was changed", "reason": "archived"}
```

- A write that wouldn't change anything (the same name, `set` to the stored value, an empty append) is refused all the same.
- The 404 and every 400 check come first.
- Drop the write, show the message, and read the tracks again.
- Creating a public link (since 3.1.0) and renaming an option (since 3.2.0) answer 423 on an archived track to every client, with or without the header.
- Without the header, ticks and notes on an archived track keep their quiet 200 or 204 with the stored state, and the track changes above are quietly ignored (200 with the unchanged track). That will end after a deprecation period.

## Retrying writes

*Since 2.3.0.* These writes aren't safe to repeat:

- `POST /api/tracks`: a retry makes a second track.
- `POST /api/ticks/toggle`: a retry flips the day back.
- `POST /api/ticks/increment`: a retry counts twice.
- `POST /api/notes/append` with `allowDuplicate`: a retry adds the line twice.
- `POST /api/ticks/set` with `appendNote` and `allowDuplicateNote`. It takes the key on every call.
- `POST /api/public-links` (since 3.1.0): a retry makes a second link.
- A Tick only public link's call (since 3.1.0), whatever its method.

They take an optional `Idempotency-Key` header, from the IETF draft [draft-ietf-httpapi-idempotency-key-header](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header). Needs `features.idempotencyKey`.

```
Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"
```

- **The key** is a new UUID for each change, sent in double quotes (a Structured Fields string). The quotes are optional, and letter case doesn't matter. Anything else answers 400 `invalidValue`, `Idempotency-Key must be a UUID`, and nothing is written.
- **On a retry**, send the same key with exactly the same parameters. Keep the key with the queued change until an answer arrives.
- **A repeat** (the same user, key, endpoint and parameters) gets the first request's answer, with the same status and body, and changes nothing. That includes an error the first request got, and holds even if the day has changed since.
- **The same key with other parameters, or on another endpoint**, answers 422 `idempotencyKeyReused`, `This Idempotency-Key was already used for another request`. Nothing is written.
- **A repeat while the first request is still running** answers 409 `requestInProgress`, `This request is still being handled; try again shortly`.
- **Keys are kept for 30 days.** A retry after that is handled as a new request.
- If the server failed partway through the first request, so no answer was stored, a repeat more than 5 minutes later is handled as a new request.
- Without the header, every endpoint works as before. A server without `idempotencyKey` ignores the header, so a retried write may happen twice there.

## Tracks

### The track object

```json
{ "id": 3, "name": "Exercise", "type": "boolean",
  "sortOrder": 2, "private": false, "archivedAt": null,
  "publicLinkCount": 0,
  "unit": null, "decimalPlaces": null, "allowNegative": null,
  "options": null, "orderedScale": null }
```

A Number track and a Choice track:

```json
{ "id": 7, "name": "Run", "type": "number", …,
  "unit": "km", "decimalPlaces": 1, "allowNegative": false,
  "options": null, "orderedScale": null }
{ "id": 8, "name": "Mood", "type": "choice", …,
  "unit": null, "decimalPlaces": null, "allowNegative": null,
  "options": [ { "value": 1, "label": "Low" }, { "value": 2, "label": "OK" },
               { "value": 3, "label": "Happy" } ],
  "orderedScale": true }
```

| Field | Type | Since | Notes |
|---|---|---|---|
| `id` | int | 1.0.0 | Assigned by the server. |
| `name` | string | 1.0.0 | Trimmed, never empty. Names may repeat. |
| `type` | string | 1.0.0 | `boolean` or `counter`; `number` and `choice` since 3.2.0. Fixed at creation. Without `includeAllTypes`, reads list only `boolean` and `counter` tracks. With it they list every type, including any added later, so a client that sends it must cope with a type it doesn't know. |
| `sortOrder` | int | 1.0.0 | Journal order, ascending. |
| `private` | bool | 1.0.0 | A display hint only. |
| `archivedAt` | date or null | 2.2.0 | The day the track was archived; `null` means active. |
| `publicLinkCount` | int | 3.1.0 | The track's live public links. 0 on an archived track, since archiving revokes them. |
| `unit` | string or null | 3.2.0 | Number: shown after the value, `""` when there is none, at most 255 characters. `null` on other types, as are the four fields below on types that don't have them. |
| `decimalPlaces` | int or null | 3.2.0 | Number: 0 to 3. Values are rounded to it when written. |
| `allowNegative` | bool or null | 3.2.0 | Number: whether values below 0 can be written. |
| `options` | list or null | 3.2.0 | Choice: 2 to 99 `{"value", "label"}`, in order. `value` is the option's place, 1 for the first, and is what a day stores. Labels are unique, ignoring case. Options can be renamed, never added, removed or reordered. |
| `orderedScale` | bool or null | 3.2.0 | Choice: the options run from lowest to highest, as on a mood or pain scale. Only a scale has a highest and lowest value in stats. |

### GET /api/tracks

*Since 1.0.0.* The user's tracks in journal order (`sortOrder`, then `id`).

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `includeArchived` | bool, default false | 2.2.0 | Also return archived tracks. |
| `includeAllTypes` | bool, default false | 3.2.0 | Also list Number and Choice tracks, and any type added later. Without it, only `boolean` and `counter` tracks. |

| Status | Answer |
|---|---|
| 200 | A list of track objects. |
| 400 | `includeArchived must be true or false` (since 3.1.0) · `includeAllTypes must be true or false` (since 3.2.0) |

- **Changed in 2.2.0:** archived tracks are left out by default, so a client that knows nothing about archiving simply stops seeing them.
- **Changed in 3.2.0:** Number and Choice tracks are left out unless `includeAllTypes=true`, so a client written before them never sees them.

### POST /api/tracks

*Since 1.0.0.* Creates a track at the end of the list. Takes an `Idempotency-Key` (since 2.3.0): a repeat answers 201 with the track the first request made.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `name` | string, required | 1.0.0 | Trimmed; empty fails. |
| `type` | string, required | 1.0.0 | `boolean` or `counter`; `number` or `choice` since 3.2.0. |
| `unit` | string | 3.2.0 | Number only. Trimmed, at most 255 characters. Empty or missing: none. |
| `decimalPlaces` | int | 3.2.0 | Number only. 0, 1, 2 or 3. Missing: 1. |
| `allowNegative` | bool | 3.2.0 | Number only. Missing: false. |
| `options` | list of strings | 3.2.0 | Choice only, required. 2 to 99 labels in order, each trimmed, not blank, at most 255 characters, and no two the same ignoring case. Form-encoded: `options[]=Low&options[]=OK`. |
| `orderedScale` | bool | 3.2.0 | Choice only. Missing: false. |

A setting for another type is 400, so Yes/No and counter tracks take none. The checks run in this order, before anything is written: name, type, settings for another type, the type's own settings, the track limit. The settings are part of an `Idempotency-Key`'s parameters.

| Status | Answer |
|---|---|
| 201 | The new track object. |
| 400 | `Track name cannot be empty` · `Invalid track type: x` · `Track names can be at most 255 characters` (since 2.3.0) |
| 400 | Since 3.2.0: `unit, decimalPlaces and allowNegative are only for number tracks` · `options and orderedScale are only for choice tracks` · `Choice tracks need 2 to 99 options` · `Options must be a list of labels` · `Option labels cannot be empty` · `Option labels can be at most 255 characters` · `Two options can't have the same label: "Happy"` · `Decimal places must be 0, 1, 2 or 3` · `Units can be at most 255 characters` · `allowNegative must be true or false` |
| 403 | `Maximum of 99 tracks reached`. Archived tracks count. |
| 409 | `requestInProgress` (since 2.3.0) |
| 422 | `idempotencyKeyReused` (since 2.3.0) |

### PUT /api/tracks/{id}

*Since 1.0.0.* Changes a track. Every parameter is optional; the ones left out stay as they are. `type` is ignored if sent.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `name` | string | 1.0.0 | Trimmed, not empty, at most 255 characters. |
| `sortOrder` | int | 1.0.0 | Usually set through reorder instead. |
| `private` | bool | 1.0.0 | |
| `archived` | bool | 2.2.0 | `true` archives the track. `false` restores it and moves it to the end of the list; on an active track it does nothing. |
| `archivedAt` | date | 2.2.0 | The client's today, read only when archiving. Missing: the server's date. |
| `unit` | string | 3.2.0 | Number only. The new unit; empty removes it. |
| `decimalPlaces` | int | 3.2.0 | Number only. 0 to 3. Stored values don't change: with fewer places, older days show rounded. |
| `allowNegative` | bool | 3.2.0 | Number only. Applies to new values; days already stored keep theirs. |
| `orderedScale` | bool | 3.2.0 | Choice only. |

- A setting for another type is 400. Options aren't changed here: see `PUT /api/tracks/{id}/options/{value}`.
- Parameters can be combined. `name`, `private` and `sortOrder` apply first, then `archived`, so a restore's end-of-list place wins over a `sortOrder` in the same request.
- An archived track only takes a restore (`archived=false`). A new `name`, `private`, `sortOrder` or `archived=true` changes nothing: 200 with the unchanged track, or 423 with `Prefer: handling=strict`. Re-archiving keeps the first date. (Since 2.3.0.)
- Archiving revokes the track's public links, and restoring brings none back (since 3.1.0).

| Status | Answer |
|---|---|
| 200 | The updated track object. |
| 400 | `Track name cannot be empty` · `Track names can be at most 255 characters` (since 2.3.0) · `Archive date must be a date as YYYY-MM-DD` (since 2.2.0) |
| 400 | `private must be true or false` · `archived must be true or false` (since 3.1.0). Nothing changed. |
| 400 | The settings' messages, as when creating a track (since 3.2.0). Nothing changed. |
| 404 | `Track not found` |
| 409 | `databaseConflict`, `The tracks changed while saving; try again`: another change took the track's place in the list. Nothing changed. (Since 3.1.0.) |
| 423 | The track is archived, with `Prefer: handling=strict` (since 2.3.0). |

### PUT /api/tracks/reorder

*Since 1.0.0.* Sets the journal order.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `trackIds[]` | list of ints, required | 1.0.0 | The wanted order. Form-encoded: `trackIds[]=1&trackIds[]=5`. |
| `includeAllTypes` | bool, default false | 3.2.0 | Whether the answer lists every type, as on `GET /api/tracks`. The reorder itself always covers every track. |

- Tracks missing from `trackIds`, such as archived ones or tracks the client doesn't know about, follow the listed ones in their previous order (since 2.2.0).
- If `trackIds` lists an archived track, nothing changes: 200 with the active tracks in their current order, or 423 with `Prefer: handling=strict` (since 2.3.0).

| Status | Answer |
|---|---|
| 200 | The active tracks in their new order. |
| 400 | `trackIds is required` |
| 409 | `databaseConflict`, `The tracks changed while reordering; try again`: tracks kept being created or restored elsewhere while it ran. Nothing changed. (Since 3.1.0.) |

### PUT /api/tracks/{id}/options/{value}

*Since 3.2.0.* Renames one option of a Choice track. The option keeps its place, so every day that holds it shows the new label. Options can't be added, removed or reordered. Needs `features.choiceTracks`; older servers answer 404. Safe to repeat.

| Parameter | Type | Notes |
|---|---|---|
| `id` | int, in the path | The track. |
| `value` | int, in the path | The option's place, 1 for the first. |
| `label` | string, required | Trimmed, not blank, at most 255 characters, and not another option's label, ignoring case. |

| Status | Answer |
|---|---|
| 200 | The whole track object, also when the option already had this label. |
| 400 | `wrongTrackType`, `Only choice tracks have options` · `invalidValue`: `Option labels cannot be empty` · `Option labels can be at most 255 characters` · `Two options can't have the same label: "Happy"` |
| 404 | `trackNotFound`, `Track not found` · `optionNotFound`, `Option not found` |
| 423 | The track is archived, for every client, with or without `Prefer: handling=strict`. Checked after the label. |

### DELETE /api/tracks/{id}

*Since 1.0.0.* Deletes the track, archived or not, with all its ticks, notes, public links and a Choice track's options. This can't be undone.

| Status | Answer |
|---|---|
| 204 | Deleted, no body. |
| 404 | `Track not found` |

## Ticks

### GET /api/ticks

*Since 1.0.0.* The ticks in a date range.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `from`, `to` | date, required | 1.0.0 | Both days included. `from` = `to` reads one day. |
| `trackId` | int | 3.1.0 | Only that track's ticks, also an archived track's, of any type. Missing or 0: every track. Needs `features.trackFilter`; an older server ignores it and answers every track, so filter the answer by `trackId` yourself. |
| `includeAllTypes` | bool, default false | 3.2.0 | For every track, also Number and Choice tracks' ticks. Without it, only `boolean` and `counter` tracks'. |

```json
[ { "id": 42, "trackId": 1, "date": "2026-10-16", "value": 1 },
  { "id": 43, "trackId": 2, "date": "2026-10-16", "value": 3 } ]
```

- Newest first, and only existing ticks: a missing day means not ticked. Includes private and archived tracks' ticks.
- A Number's `value` (since 3.2.0) is a JSON number, with a point when it isn't whole (`10.4`, `-1.5`, `72`); 0 and negatives are readings. Read it as an exact decimal: it never has more than 3 decimal places.
- A Choice's `value` (since 3.2.0) is the option's place. Its label is in the track's `options`.
- To read one track's whole history, send its first day as `from` (`firstTick` from `GET /api/stats` with `trackId`, or `oldest` from `GET /api/ticks/bounds`), today as `to`, and `trackId`.

| Status | Answer |
|---|---|
| 200 | A list of ticks. |
| 400 | `From and to must be dates as YYYY-MM-DD` (missing or not a real date) · `From must be on or before to` (since 3.1.0) · `includeAllTypes must be true or false` (since 3.2.0) |
| 404 | `trackNotFound`, `Track not found`: the `trackId` isn't the user's. Checked after the dates. (Since 3.1.0.) |

### GET /api/ticks/bounds

*Since 1.1.0.* The first and last tick date per track, worked out in the database. No parameters. Needs `features.tickBounds`; older servers answer 404.

```json
[ { "trackId": 1, "oldest": "2026-03-07", "newest": "2026-10-16" } ]
```

Tracks with no ticks are left out. Private and archived tracks are included, and tracks of every type: `includeAllTypes` doesn't apply. Every stored tick counts, also one dated after today. Notes have their own bounds: `GET /api/notes/bounds`.

### POST /api/ticks/toggle

*Since 1.0.0.* Flips a Yes/No tick for one day. Not safe to repeat: a retry flips the day back. Send an `Idempotency-Key`, or use `set` with `true` or `false` instead.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `trackId` | int, required | 1.0.0 | |
| `date` | date, required | 1.0.0 | A real `YYYY-MM-DD` date (checked since 2.3.0). |
| `note` | string | 2.3.0 | Sets the day's note too; empty deletes it. Missing: the note is left alone. Needs `features.notes`. |
| `appendNote` | string | 2.3.0 | Appends to the day's note instead, as `notes/append` does. Not with `note`. Needs `features.notesAppend`. |
| `allowDuplicateNote` | bool | 2.3.0 | With `appendNote`: append even a line that's already there. |

Every check, the note's included, runs before anything is written. A 409 from `appendNote` comes after the tick is written, so retry the note alone with `notes/append`.

| Status | Answer |
|---|---|
| 200 | `{"ticked": true, "value": 1}` if the tick now exists, `{"ticked": false, "value": 0}` if it was removed. `value` since 2.3.0; read `value`, as `ticked` stays for older clients. With `note` or `appendNote`, also `note`, the whole stored note afterwards. |
| 400 | `Toggle is only available for boolean tracks` · `Date must be a date as YYYY-MM-DD` (since 2.3.0) · `allowDuplicateNote must be true or false` (since 3.1.0) |
| 404 | `Track not found` |
| 409 | `requestInProgress` (since 2.3.0) |
| 422 | `idempotencyKeyReused` (since 2.3.0) |
| 423 | The track is archived, with `Prefer: handling=strict` (since 2.3.0). Without it, nothing changes and the answer is whether a tick exists. |

### POST /api/ticks/set

*Since 1.0.0.* Sets one day's value: a counter to a number, a Yes/No tick to ticked or not (since 2.3.0), or a Number or Choice day to a value, or clears it (since 3.2.0). Safe to repeat.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `trackId` | int, required | 1.0.0 | |
| `date` | date, required | 1.0.0 | A real `YYYY-MM-DD` date (checked since 2.3.0). |
| `value` | required | 1.0.0 | Depends on the track's type; see below. |
| `note` | string | 2.3.0 | Sets the day's note too; empty deletes it. Missing: the note is left alone. Needs `features.notes`. |
| `appendNote` | string | 2.3.0 | Appends to the day's note instead, as `notes/append` does. Not with `note`. Needs `features.notesAppend`. |
| `allowDuplicateNote` | bool | 2.3.0 | With `appendNote`: append even a line that's already there. |

The `value`, by type:

- **Counter:** a whole number from 0 to 2147483647, as a JSON integer or a string of digits. 0 deletes the tick. A negative, missing or non-whole value (`-1`, `5abc`, `5.0`, `true`) is 400 (since 3.1.0; before, it was stored as 0, cut to a number, or deleted the tick if negative).
- **Yes/No** (since 2.3.0, `features.booleanSet`): `true` or `false`, or any boolean as in Conventions.
- **Number** (since 3.2.0): a number with a point and no grouping (`10.4`, `-3`), as a JSON number or a string, from −2147483.647 to 2147483.647. It is rounded half away from zero to the track's decimal places. 0 is a reading and is stored. A value below 0 needs `allowNegative`.
- **Choice** (since 3.2.0): an option's place, from 1 to the number of options.
- **Number and Choice:** `null` or `""` clears the day. Leaving `value` out is 400. With an `Idempotency-Key`, a `value` left out and a `null` are different requests.

Every check, the note's included, runs before anything is written. A 409 from `appendNote` comes after the value is written, so retry the note alone with `notes/append`. `set` takes absolute values, so two devices adding to the same counter offline both send the same total and one addition is lost: use `increment` to add.

| Status | Answer |
|---|---|
| 200 | `{"value": 3}`, the stored value. On a counter 0 means no row; on Yes/No it is 1 or 0 (since 2.3.0); on Number and Choice it is the value as the tick list writes it, or `null` for a cleared day (since 3.2.0). With `note` or `appendNote`, also `note`. |
| 400 | `Date must be a date as YYYY-MM-DD` (since 2.3.0) · `Yes/no tracks take a value of true or false` (since 2.3.0) |
| 400 | `Counter tracks take a whole number, 0 or more` · `Counter values can be at most 2147483647` · `allowDuplicateNote must be true or false` (since 3.1.0) |
| 400 | `Number tracks take a number, such as 10.4` (also when `value` is left out) · `Numbers must be between -2147483.647 and 2147483.647` · `This track doesn't allow negative values` · `Choice tracks take the number of an option, from 1 to 4` (since 3.2.0) |
| 404 | `Track not found` |
| 409 | `requestInProgress` (since 2.3.0) |
| 422 | `idempotencyKeyReused` (since 2.3.0) |
| 423 | The track is archived, with `Prefer: handling=strict` (since 2.3.0). Without it, nothing changes and the answer is the stored value. |

### POST /api/ticks/increment

*Since 2.3.0.* Adds to a counter atomically, so additions from any mix of devices and scripts all count. Needs `features.counterIncrement`. Not safe to repeat: a retry adds twice, so send an `Idempotency-Key`.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `trackId` | int, required | 2.3.0 | |
| `date` | date, required | 2.3.0 | A real `YYYY-MM-DD` date. |
| `delta` | int, required | 2.3.0 | Not 0; negative allowed. The value never goes below 0: at 0 or less, the tick is deleted. A JSON integer or a string of digits with an optional minus; `1.9` or `5abc` is 400 (since 3.1.0). A result above 2147483647 is 400 with nothing written (since 3.1.0). A delta below −2147483647 counts as −2147483647. |
| `note` | string | 2.3.0 | As on `set`. |
| `appendNote` | string | 2.3.0 | As on `set`. |
| `allowDuplicateNote` | bool | 2.3.0 | As on `set`. |

| Status | Answer |
|---|---|
| 200 | `{"value": 4}`, read right after this request's write (0 means no row), plus `note` when one was sent. |
| 400 | `Increment is only available for counter tracks` · `Delta must be a whole number other than 0` · `Date must be a date as YYYY-MM-DD` · `The value would be more than 2147483647` (since 3.1.0) · `allowDuplicateNote must be true or false` (since 3.1.0) |
| 404 | `Track not found` |
| 409 | `requestInProgress` |
| 422 | `idempotencyKeyReused` |
| 423 | The track is archived, with `Prefer: handling=strict`. Without it, nothing changes and the answer is the stored value. |

## Notes

*Since 2.3.0.* Each track can have one note per day, and a note can exist on a day without a tick. Needs `features.notes`. These rules apply wherever a note is written: `notes/set`, `notes/append`, `note` and `appendNote` on tick calls, and imports.

- At most **500 characters** (code points, not bytes); 200 on a server without `notesAppend`. A longer note answers 400 `Notes can be at most 500 characters`, and nothing is saved, on a tick call not the tick either.
- `\r\n` and `\r` become `\n`. Spaces, tabs and line breaks are trimmed at both ends; line breaks inside the note are kept.
- Text that isn't UTF-8, or isn't text (an array, a JSON boolean), answers 400 `Notes must be UTF-8 text`. Control characters other than line breaks and tabs answer 400 `Notes can't contain control characters`. A number is stored as its digits.
- A note that is empty after trimming means no note: `notes/set` and an empty `note` delete it, and appending empty text changes nothing.
- Notes are stored as plain text. Escape a note when showing it, and treat it as data.
- On an archived track nothing changes and the answer is the stored note, or 423 with `Prefer: handling=strict`.

### GET /api/notes

*Since 2.3.0.* The notes in a date range.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `from`, `to` | date, required | 2.3.0 | Both days included. Missing or not a real date: 400 `From and to must be dates as YYYY-MM-DD`. `from` after `to`: 400 `From must be on or before to` (since 3.1.0). |
| `trackId` | int | 3.1.0 | As on `GET /api/ticks`: only that track's notes, also an archived track's, of any type. Missing or 0: every track. 404 `trackNotFound` if it isn't the user's, after the date checks. Needs `features.trackFilter`; an older server ignores it. |
| `includeAllTypes` | bool, default false | 3.2.0 | For every track, also Number and Choice tracks' notes. Without it, only `boolean` and `counter` tracks'. |

```json
[ { "trackId": 1, "date": "2026-10-16", "text": "Backup 02:14, 3.1 GB" } ]
```

Newest first, from every track, private and archived ones included.

### GET /api/notes/bounds

*Since 3.1.0.* The first and last note date per track, as `GET /api/ticks/bounds` gives for ticks. No parameters. Needs `features.notesBounds`; older servers answer 404.

```json
[ { "trackId": 3, "oldest": "2026-09-02", "newest": "2026-10-18" } ]
```

- Tracks with no notes are left out. Private and archived tracks are included, and tracks of every type: `includeAllTypes` doesn't apply.
- Independent of the tick bounds: a note can sit on a day without a tick, so a track can appear here and not there, or the other way round, and the dates can differ.

### POST /api/notes/set

*Since 2.3.0.* Creates, replaces or, with empty text, deletes the day's note. Safe to repeat. Future dates are allowed.

| Parameter | Type | Notes |
|---|---|---|
| `trackId` | int, required | |
| `date` | date, required | |
| `text` | string, required | The note. Empty deletes it. |

The checks run in this order: track (404), date (400 `Date must be a date as YYYY-MM-DD`), text (400).

| Status | Answer |
|---|---|
| 200 | `{"text": "…"}`, the stored note, or `""` after a delete. |
| 400 | The date and text errors above. |
| 404 | `Track not found` |
| 423 | The track is archived, with `Prefer: handling=strict`. |

### POST /api/notes/append

*Since 2.3.0.* Adds text to the day's note on a new line, worked out on the server so two writers can't overwrite each other. Needs `features.notesAppend`. Safe to repeat, because a line that's already there is skipped. With `allowDuplicate` it isn't, so send an `Idempotency-Key`.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `trackId`, `date`, `text` | required | 2.3.0 | As for `notes/set`. |
| `allowDuplicate` | bool, default false | 2.3.0 | Append even when the text is already there. |

- With no note yet, it stores the text. Otherwise it stores the note, a line break, then the text.
- If every line of the text already appears, in order, as whole consecutive lines of the note, nothing changes and the call succeeds. This is case-sensitive, and part of a line doesn't count.
- Empty text changes nothing and deletes nothing.

| Status | Answer |
|---|---|
| 200 | `{"text": "…"}`, the whole stored note afterwards. |
| 400 | `The note would be longer than 500 characters` (nothing written) · the date and text errors of `notes/set` · `allowDuplicate must be true or false` (since 3.1.0) |
| 404 | `Track not found` |
| 409 | `noteConflict`, `The note changed while appending; try again`: another writer won three times in a row · `requestInProgress` |
| 422 | `idempotencyKeyReused` |
| 423 | The track is archived, with `Prefer: handling=strict`. |

### DELETE /api/notes

*Since 2.3.0.* Deletes the day's note. Query parameters `trackId` and `date`. Answers 204, also when there was no note, so a retry never fails. 404 for a track that isn't the user's, 400 for a bad date.

## Stats

### GET /api/stats

*Since 2.3.0.* The numbers Tickbuddy shows on Analytics, per track, worked out by the server so every client shows the same. Needs `features.stats`; `from` and `to` need `features.statsPeriod`.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `today` | date | 2.3.0 | The client's date. Missing: the server's. A bad date: 400 `Today must be a date as YYYY-MM-DD`. |
| `from`, `to` | date | 2.3.0 | A period; either may be sent alone. A bad date: 400 `From and to must be dates as YYYY-MM-DD`. `from` after `to`: 400 `From must be on or before to`. An older server without `statsPeriod` ignores both and answers all-time numbers. |
| `trackId` | int | 2.3.0 | One track, archived ones too, of any type. The answer is still a list. 404 if it isn't the user's. |
| `includeRuns` | bool | 2.3.0 | Add every streak and break as `runs`. |
| `includeArchived` | bool | 2.3.0 | Include archived tracks when listing every track. |
| `includeAllTypes` | bool | 3.2.0 | When listing every track, also Number and Choice tracks. |

Each track counts from a start day `S` to an end day `E`. `E` is the earliest of `to`, `today` and an archived track's archive date. `S` is `from`, or else the track's first tick. Streaks and breaks that cross an edge are cut there. With `from`, unticked days at the start count as a break, and the weekly mean is over the whole period. Every track gets an entry, also one with no ticks.

```json
[ { "trackId": 3, "total": 412, "weeklyMean": 9.58,
    "last7": 11, "previous7": 14, "peak7": 31,
    "current": { "isStreak": true, "length": 6, "from": "2026-10-11", "to": "2026-10-16" },
    "upToYesterday": true,
    "longestStreak": { "isStreak": true, "length": 41, "from": "2026-03-02", "to": "2026-04-11" },
    "longestBreak": { "isStreak": false, "length": 9, "from": "2026-07-20", "to": "2026-07-28" },
    "firstTick": "2025-12-21", "lastTick": "2026-10-16",
    "maxValue": 7, "minValue": 0 } ]
```

| Field | Meaning |
|---|---|
| `total` | The sum of the values: ticked days on a Yes/No track, days with an entry on a Number or Choice track. |
| `weeklyMean` | `total` over the weeks from `S` to `E`, at least one week. Not rounded. |
| `last7`, `previous7` | The last 7 days up to `E`, and the 7 before them. |
| `peak7` | The best 7-day sum. |
| `current` | The last streak or break as of `E`. |
| `upToYesterday` | The streak stops the day before an open `E`, so it can still continue today. |
| `longestStreak`, `longestBreak` | The first of several equally long ones. |
| `firstTick`, `lastTick` | Within `S` to `E`. |
| `maxValue`, `minValue` | The highest and lowest value, over every day, an unticked day counting 0. |
| `runs` | With `includeRuns`: every streak and break. |

**Number and Choice tracks** (since 3.2.0): each day with an entry counts 1 in `total`, `weeklyMean`, the 7-day sums and the streaks, whatever its value, a Number's 0 or negative value included. `maxValue` and `minValue` are the highest and lowest stored value among the days with an entry (a Number's as a decimal, a Choice's as a place), `null` when there is none, and always `null` on a Choice track that isn't an ordered scale.

## Public links

*Since 3.1.0.* A public link is an address that ticks or reads one track without signing in, for scripts and home automations. Needs `features.publicLinks`; older servers answer 404 on the routes.

### The public link object

```json
{ "id": 12, "trackId": 3, "name": "Home Assistant",
  "permission": "tick", "allowedDays": "activeAndPrevious",
  "url": "https://cloud.example.com/ocs/v2.php/apps/tickbuddy/ping/…",
  "lastUsedAt": 1795250400 }
```

| Field | Type | Notes |
|---|---|---|
| `id` | int | Assigned by the server. |
| `trackId` | int | The one track it ticks or reads. |
| `name` | string | 1 to 255 characters once trimmed. Names may repeat. |
| `permission` | string | `tick` (Tick only) or `view` (View only). Fixed at creation. |
| `allowedDays` | string | `activeDay`, `activeAndPrevious`, `oneWeek` or `allDays`: the days a call may name, as the web app's editable days (Today, +1 day, 1 week, All), never after today in the owner's time zone. Fixed at creation. |
| `url` | string | The whole address, built from the server's own settings. Its 32-character token is the credential, so treat the address as a password. |
| `lastUsedAt` | int or null | Unix seconds of the last call that reached the link, noted at most once a minute. `null` until the first. |

- Revoked links are never listed, and a revoked link can't be brought back.
- Archiving a track revokes its links, and restoring it brings none back. Deleting the track, or an import with `replace`, deletes them.
- A track has at most 99 live links.

### GET /api/public-links

*Since 3.1.0.* The user's live links, oldest first. Query parameter `trackId` (int) for one track's links, also an archived track's (which has none); missing or 0 for every track's.

| Status | Answer |
|---|---|
| 200 | A list of public link objects. |
| 404 | `trackNotFound`, `Track not found` |

### POST /api/public-links

*Since 3.1.0.* Creates a link. Takes an `Idempotency-Key`, so a retry doesn't make a second link.

| Parameter | Type | Notes |
|---|---|---|
| `trackId` | int, required | |
| `name` | string, required | Trimmed, 1 to 255 characters. |
| `permission` | string, required | `tick` or `view`. |
| `allowedDays` | string, required | `activeDay`, `activeAndPrevious`, `oneWeek` or `allDays`. |

| Status | Answer |
|---|---|
| 201 | The new link, with its `url`. |
| 400 | `Link name cannot be empty` · `Link names can be at most 255 characters` · `Unknown value "x" for permission` · `Unknown value "x" for allowedDays` |
| 403 | `publicLinkLimitReached`, `Maximum of 99 public links reached` |
| 404 | `Track not found` |
| 409 | `requestInProgress` |
| 422 | `idempotencyKeyReused` |
| 423 | The track is archived, for every client: `Exercise is archived, so nothing was changed`. |

### PUT /api/public-links/{id}

*Since 3.1.0.* Renames a link, with the parameter `name`, trimmed, 1 to 255 characters. Only the name can change; `permission` and `allowedDays` are ignored if sent. Safe to repeat.

| Status | Answer |
|---|---|
| 200 | The renamed link. |
| 400 | `Link name cannot be empty` · `Link names can be at most 255 characters` |
| 404 | `publicLinkNotFound`, `Public link not found`: not the user's, or revoked. |

### DELETE /api/public-links/{id}

*Since 3.1.0.* Revokes a link: from then on its address answers 404. The ticks and notes it made are kept. Answers 204, also when the link was already revoked, so a retry never fails; 404 `publicLinkNotFound` if it isn't the user's.

### GET, POST or HEAD /ping/{token}

*Since 3.1.0.* The address in `url`, which anything that can open a web address can call. Send no `OCS-APIRequest`, no `Authorization` and no CSRF token: the token is the credential, and a signed-in browser's session grants nothing. GET, POST and HEAD all do what the link is for, so a chat app's link preview would call it. Answers the usual OCS envelope as JSON, unless asked for XML (`?format=xml` or `Accept: application/xml`).

| Parameter | Link | Notes |
|---|---|---|
| `value` | Tick only | Yes/No: `true` or `false`; missing: true, so a second call on the same day changes nothing. Counter: the day's new value, a whole number from 0 to 2147483647 (0 clears it); missing: adds 1. Number (since 3.2.0), required: the day's value with a point (`value=72.4`), rounded to the track's decimal places. Choice (since 3.2.0), required: an option's place. A link never clears a Number or Choice day. |
| `date` | Both | `YYYY-MM-DD`. Missing: today in the owner's Nextcloud time zone. Must be one of the link's allowed days. Not with `from` and `to`. |
| `appendNote` | Tick only | Text added to the day's note on a new line; a line already there is skipped. Or send it as a POST body with `Content-Type: text/plain`, but not both. |
| `from`, `to` | View only | A range instead of `date`. Both ends must be allowed days. |
| `includeNotes` | View only | bool: also answer the notes. |

```
Tick only:  { "value": 1 }                     (+ "note" when text was appended)
View only:  { "ticks": [ { "id": 42, "trackId": 3, "date": "2026-11-17", "value": 1 } ],
              "notes": [ { "trackId": 3, "date": "2026-11-17", "text": "…" } ] }
```

| Status | Answer |
|---|---|
| 200 | Tick only: the day's value afterwards (1 or 0 on Yes/No, a Number's decimal, a Choice's place). View only: the ticks as `GET /api/ticks` gives them, newest first, and `notes` only with `includeNotes`. |
| 400 | `invalidValue`: a bad parameter, or the other kind of link's parameters (`Tick only links don't read: leave out from, to and includeNotes`, `View only links don't tick: leave out value, appendNote and the body`), `Send date, or from and to, not both`, `Send appendNote or a text body, not both`, `Number tracks need a value, such as value=10.4` (since 3.2.0), `Choice tracks need a value, the number of an option from 1 to 4` (since 3.2.0). `wrongTrackType`: a track type links can't tick. Nothing was done. |
| 403 | `dayNotAllowed`, such as `This link's allowed days (+1 day) don't include 2026-11-10`. Nothing was done. |
| 404 | `publicLinkNotFound`: an unknown or revoked link, or an owner who can't sign in (disabled, deleted, or without Tickbuddy). |
| 409 | `noteConflict`: the tick was written, but the note kept changing · `requestInProgress` |
| 422 | `idempotencyKeyReused` |
| 423 | The track was archived during the call, only with `Prefer: handling=strict`. |
| 429 | Nextcloud's own answer: more than 60 calls a minute from one address, all links together (an empty body; an admin can change the limit with `ratelimit_overwrite`), or brute-force protection after repeated guesses (XML). |

- A Tick only link's calls honour `Idempotency-Key` as the API's writes do, keyed to the link's owner, whatever the method.
- Only a token that was never a link counts as a guess for Nextcloud's brute-force protection. A revoked link's 404 doesn't, so a script still calling one doesn't slow down the other links called from the same server.

## Preferences

### GET /api/preferences

*Since 1.0.0.* The web app's toolbar choices, one value for desktop and one for mobile. Only the web app reads them; other clients keep their own.

```json
{ "journalLayoutDesktop": "daysDown", "journalLayoutMobile": "daysDown",
  "showPrivateDesktop": false, "showPrivateMobile": false,
  "editableDaysDesktop": "allDays", "editableDaysMobile": "activeDay" }
```

| Field | Values | Starts as |
|---|---|---|
| `journalLayoutDesktop`, `journalLayoutMobile` | `daysDown`, `tracksDown` | `daysDown` |
| `showPrivateDesktop`, `showPrivateMobile` | bool | `false` |
| `editableDaysDesktop`, `editableDaysMobile` | `activeDay`, `activeAndPrevious`, `oneWeek`, `allDays` | `allDays` on desktop, `activeDay` on mobile |

**Changed in 3.0.0:** these six fields replace `defaultView`. A stored value that is no longer allowed reads as the starting value.

### PUT /api/preferences

*Since 1.0.0.* Any of the six fields above; only the ones sent change. Every value is checked before anything is saved. Answers every preference. Since 3.0.0, `defaultView` is no longer read: a client that sends it changes nothing and gets 200.

| Status | Answer |
|---|---|
| 200 | Every preference. |
| 400 | `Unknown value "x" for journalLayoutDesktop`, and the same for each field that takes a name (since 3.0.0). Nothing saved. |
| 400 | `showPrivateDesktop must be true or false`, and the same for `showPrivateMobile` (since 3.1.0). Nothing saved. |

## Backup

### GET /api/export

*Since 1.0.0.* The whole journal as a backup file, format version 1. Ticks refer to tracks by name, and ids are left out, so the file moves between servers.

| Parameter | Type | Since | Notes |
|---|---|---|---|
| `includePrivate` | bool, default false | 1.0.0 | Include private tracks, with their ticks and notes. |

```json
{ "version": 1, "exportedAt": "2026-10-17T15:30:00+00:00",
  "tracks": [ { "name": "Exercise", "type": "boolean", "sortOrder": 1, "private": false,
                "archivedAt": "2026-03-02" } ],
  "ticks":  [ { "track": "Exercise", "trackIndex": 0, "date": "2026-03-01", "value": 1 } ],
  "notes":  [ { "track": "Exercise", "date": "2026-03-01", "text": "…" } ] }
```

- `exportedAt` is in the server's time zone.
- `archivedAt` (since 2.2.0) is written only on archived tracks, which are always exported.
- `trackIndex` (since 2.2.0), the track's position in `tracks`, is written on a tick or note only when that track's name repeats in the file.
- `notes` (since 2.3.0) comes after `ticks`, only when there are notes: `{"track", "trackIndex", "date", "text"}`.
- Number tracks (since 3.2.0) carry `unit`, `decimalPlaces` and `allowNegative`, and Choice tracks carry `options` (the labels, in place order) and `orderedScale`, each only on tracks of that type. A Number tick's `value` is its decimal, a Choice tick's the option's place. The format is still version 1, so a server older than 3.2.0 refuses such a file on its unknown type.
- A file without the newer parts is byte for byte the same as an older server's.

### POST /api/import/json

*Since 1.0.0.* Imports a Tickbuddy backup. A multipart upload with `file` (at most 20 MB) and `mode`:

- `replace` deletes every track, tick and public link first.
- `merge` adds the file's tracks as new tracks, renaming clashes to `Name (imported)`.

An import runs in one transaction, so nothing is written when it fails. Every check runs before anything is written: a tick or note with a bad date, a track name that isn't in the file, a bad value, the same track and day twice, or a malformed entry fails the whole import with a message. A backup that Tickbuddy exported always passes. At most 10 imports per user in 10 minutes (since 3.1.0).

- A Yes/No value must be 1, and a counter's a whole number from 0 to 2147483647.
- A track's `private` is read as the API's booleans are: missing or `null` is false (since 3.1.0).
- A Number value (since 3.2.0) must be a JSON number with up to 3 decimal places within the limits, and is taken as stored, not rounded. A Choice value must be one of the track's places. The track settings are checked as when creating a track.

| Status | Answer |
|---|---|
| 200 | `{"tracks": 4, "ticks": 812, "notes": 30}`. `notes` since 2.3.0. |
| 400 | `No file uploaded or upload error` · `File too large (max 20 MB).` · `Invalid mode. Use "replace" or "merge".` · `Invalid JSON file` · `Unsupported backup version` · `Invalid backup: missing tracks` · more than 99 tracks · a message naming the entry that failed, such as `Track "Mood" has the option "Happy" twice` · `The import failed, so nothing was imported` (since 3.1.0) |
| 429 | More than 10 imports in 10 minutes (since 3.1.0). Nextcloud's own answer, with an empty body that isn't OCS JSON, so read the status, not the body. Wait, then try again. |

### POST /api/import

*Since 1.0.0.* The same upload, with an Android Tickmate `.db` backup. Tickmate's disabled tracks are imported as archived tracks (since 2.3.0; skipped before). At most 200,000 ticks (since 3.1.0). The limit of 10 imports in 10 minutes is counted separately from the JSON import's.

Extra 400 messages: `Not a valid Tickmate backup: missing tracks or ticks table` · `No tracks found in the backup file` (since 2.3.0) · `Not a valid Tickmate backup: the file could not be read` · `Not a valid Tickmate backup: unexpected table structure` · `The backup contains N ticks, which exceeds the limit of 200000.` (since 3.1.0)

## Known limits

- `toggle` flips rather than sets, so a retried request undoes itself. Send an `Idempotency-Key`, or use `set` with `true` or `false`.
- `set` takes absolute values, so concurrent additions to one counter can be lost. Use `increment`, with an `Idempotency-Key`, to add.
- Two writes racing on the first tick of a day can both try to insert, and one hits the unique index.
- Nothing pushes changes, so clients poll. There is no "changed since" query yet (`syncDelta` is reserved).

## Changes by release

Releases not listed changed only the web app or Nextcloud support.

### 3.2.0 (22 November 2026)

- Track types `number` (a reading a day, with `unit`, `decimalPlaces` and `allowNegative`) and `choice` (one of 2 to 99 options a day, with `options` and `orderedScale`). The five fields are on every track, `null` where the type has none. Flags `numberTracks` and `choiceTracks`.
- `includeAllTypes` on `GET /api/tracks`, `/api/ticks`, `/api/notes`, `/api/stats` and the reorder answer. Without it, reads leave the new types out, so a client written before them never meets them.
- `set` writes a Number (a decimal with a point, rounded to the track's decimal places) or a Choice (an option's place). `null` or `""` clears the day; leaving `value` out is 400. A Number's 0 is a reading and is stored.
- `PUT /api/tracks/{id}/options/{value}` renames an option. New reason `optionNotFound`.
- Stats count a Number or Choice day as one entry. `maxValue` and `minValue` come from days with an entry, and are `null` on a Choice track that isn't an ordered scale.
- Public links tick Number and Choice tracks with a `value`, which they need.
- Export and import carry both types, their settings and values. The format stays version 1, so an older server refuses such a file on its unknown type.

### 3.1.0 (17 November 2026)

- Public links: `/api/public-links` lists, creates, renames and revokes them, and `/ping/{token}` ticks or reads one track without signing in. Tracks gain `publicLinkCount`. Flag `publicLinks`; new reasons `publicLinkNotFound`, `publicLinkLimitReached` and `dayNotAllowed`.
- Archiving a track revokes its public links, and deleting it or an import with `replace` deletes them.
- `GET /api/ticks` and `GET /api/notes` take `trackId`, for one track's rows. Flag `trackFilter`.
- `GET /api/notes/bounds`: the first and last note date per track. Flag `notesBounds`.
- Boolean parameters take `true`/`false`, `1`/`0`, `on`/`off` or `yes`/`no`; anything else is 400. Before, `no` and `off` read as true.
- `set` on a counter takes a whole number from 0 to 2147483647. A negative, missing or non-whole value is 400.
- `increment` takes a whole-number `delta` only, and a result past 2147483647 is 400 with nothing written.
- `GET /api/ticks` checks `from` and `to` as notes do. `from` after `to` is 400 on ticks and on notes.
- A 409 on a track change or reorder, and a failed import, answer a fixed sentence instead of the database's own text.
- Imports: at most 10 in 10 minutes per user (429), and at most 200,000 ticks in a Tickmate backup. A backup's `private` is read as the booleans are, and a counter value above 2147483647 fails the import.

### 3.0.0 (30 October 2026)

- Preferences hold the web app's toolbar choices, one value for desktop and one for mobile: journal layout, show private tracks, editable days. `defaultView` is gone, and an unknown value is 400.

### 2.3.0 (24 October 2026)

- `POST /api/ticks/set` also sets Yes/No tracks (`value` true or false) and answers 1 or 0. Flag `booleanSet`. `toggle` answers `value` (1 or 0) as well as `ticked`.
- `POST /api/ticks/increment`, atomic. Flag `counterIncrement` becomes true.
- `GET /api/stats`: totals, weekly mean, streaks and breaks per track, worked out by the server. Flag `stats`; `from` and `to` for a period, flag `statsPeriod`.
- Notes: `GET`, `POST set`, `POST append` and `DELETE /api/notes`; `note` and `appendNote` on `toggle`, `set` and `increment`; a `notes` key in backups. Flags `notes` and `notesAppend`. At most 500 characters.
- `toggle` and `set` refuse a date that isn't a real `YYYY-MM-DD` (400). Track names over 255 characters, and a blank name on rename, answer 400 instead of 500 or 409.
- Import is stricter: bad tick dates, unknown track names, bad values and repeated days fail the whole import, and nothing is written.
- Every error carries a `reason` next to its `message`.
- `Prefer: handling=strict`: a change to an archived track answers 423 instead of a quiet 200. With or without it, renaming, changing private, re-archiving or reordering an archived track no longer changes it.
- `Idempotency-Key` on creating a track, `toggle`, `set`, `increment` and `notes/append`, so they can be retried safely. Flag `idempotencyKey`.
- Tickmate import brings in disabled tracks as archived instead of skipping them.

### 2.2.0 (17 October 2026)

- Archived tracks: `GET /api/tracks` leaves them out unless `includeArchived=true`, tracks gain `archivedAt`, and `PUT /api/tracks/{id}` takes `archived` and `archivedAt`. Flag `archivedTracks`.
- `toggle` and `set` on an archived track change nothing and answer the stored state.
- Reorder: tracks left out of `trackIds` follow the listed ones instead of jumping to the top, and the answer lists active tracks only.
- Backup files gain `archivedAt` on archived tracks and `trackIndex` on ticks of tracks whose name repeats, and import reads both.

### 1.1.0 (29 July 2026)

- `GET /api/ticks/bounds`, with flag `tickBounds`.

### 1.0.6 (5 July 2026)

- Capabilities: `data.capabilities.tickbuddy` with `version`, `apiVersion: 1` and `features`. Older servers have no `tickbuddy` key.

### 1.0.0 (25 April 2026)

- Tracks, ticks, preferences, JSON export and import, and Tickmate import.
