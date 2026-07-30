---
name: leo-labs-task-radar
description: Request priority tasking of the LeoLabs radar network for a catalog object or a named instrument, then retrieve the measurements the task produced. This skill performs write operations that commit physical radar capacity.
api: leo-labs:platform
operations:
  - searchCatalogObjects
  - getCatalogObject
  - listInstruments
  - createCatalogObjectTask
  - createInstrumentTask
  - listInstrumentTasks
  - getInstrumentTask
  - listInstrumentTaskMeasurements
  - getInstrumentStatistics
generated: '2026-07-19'
method: generated
source: openapi/leo-labs-platform-openapi.yml
---

# Task the LeoLabs radar network

Use this skill to request that LeoLabs' radars observe a specific object, or to book a window
on a specific instrument, and then collect the resulting measurements.

## Read this first — this skill writes

`createCatalogObjectTask` and `createInstrumentTask` commit **physical radar capacity** on a
real sensor network. They are **not known to be idempotent**: LeoLabs documents no idempotency
key, and each POST returns a new task `id`, so a retried or duplicated request is expected to
create a second task.

Therefore:

- **Never call either write operation without explicit user confirmation** of the object,
  the instrument (if any), and the exact time window.
- **Never blind-retry a POST.** If a create times out or errors ambiguously, call
  `listInstrumentTasks` first to check whether the task already landed, then decide.
- Echo the returned task `id` back to the user immediately.

## Steps

1. **Identify the target.** Resolve a NORAD number with `searchCatalogObjects` if needed, then
   confirm with `getCatalogObject`. Show the user `name`, `catalogNumber` and
   `noradCatalogNumber` before proposing any tasking.

2. **Choose object-first or instrument-first tasking.**
   - *Object-first* (`createCatalogObjectTask`): "track this object in this window" — let
     LeoLabs pick the radar. This is the usual choice.
   - *Instrument-first* (`createInstrumentTask`): "use this specific radar in this window."
     Call `listInstruments` first to get the available site ids (e.g. `pfisr`, `msr`) and
     their latitude, longitude, altitude, transmit frequency and power, so you can justify
     the choice. `getInstrumentStatistics` (accepts the literal `all`) helps gauge load.

3. **Confirm the window.** Both create operations require `startTime` and `endTime` in ISO
   8601 UTC. Restate the window in the user's local time as well as UTC before submitting —
   time-zone mistakes here waste real sensor time.

4. **Submit the task.** POST the create operation with an
   `application/x-www-form-urlencoded` body containing `startTime` and `endTime`
   (`createCatalogObjectTask` also accepts `priority`; the official LeoLabs client sends
   `100`). The response is `{"id": <taskId>}`.

5. **Follow the task.** Use `listInstrumentTasks` and `getInstrumentTask` to check state.

6. **Collect the data.** Once the task has run, call `listInstrumentTaskMeasurements` with the
   instrument id and task id. Read the `corrected` observables for analysis; `values` holds
   the raw radar returns and `corrections[]` the audit trail of what was applied.

## Failure handling

- `401` means the key pair is missing, invalid, or was base64-encoded by mistake. The
  Authorization value is the literal `basic ` followed by `accessKey:secretKey`, unencoded.
- Errors are plain JSON with a top-level `error` member; there is no RFC 9457 problem detail
  and no published error-code registry, so surface the raw message to the user.
- The official client retries non-2xx/3xx responses up to 5 times with no backoff. **Do not
  copy that behaviour for the two create operations** — see the idempotency warning above.
