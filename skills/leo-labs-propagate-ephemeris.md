---
name: leo-labs-propagate-ephemeris
description: Propagate a LeoLabs orbital state vector forward or backward in time to produce an ephemeris with covariance, for conjunction screening and pass planning.
api: leo-labs:platform
operations:
  - searchCatalogObjects
  - listCatalogObjectStates
  - getCatalogObjectStatePropagation
  - listCatalogObjectPlannedPasses
generated: '2026-07-19'
method: generated
source: openapi/leo-labs-platform-openapi.yml
---

# Propagate an ephemeris from a LeoLabs state vector

Use this skill to turn a LeoLabs orbit determination into a time series of positions,
velocities and covariances — the input to conjunction screening, sensor pointing and pass
planning.

## Before you start

- Authenticate with `Authorization: basic <accessKey>:<secretKey>` (keys colon-joined,
  unencoded — not RFC 7617 HTTP Basic).
- Every operation here is a read.

## Steps

1. **Get a state to propagate from.** If you only have a NORAD number, resolve it first with
   `searchCatalogObjects`. Then call `listCatalogObjectStates` with `latest=1` and take the
   `id` and `timestamp` of the returned state.

2. **Check the horizon before you ask.** LeoLabs supports propagation up to **plus or minus
   7 days** from the state's `timestamp`. Compute your requested `startTime`/`endTime`
   relative to that timestamp, not relative to now — if the state is already three days old,
   you have four days of forward horizon left, not seven. If the request exceeds the horizon,
   tell the user and offer to re-run against a fresher state rather than silently truncating.

3. **Propagate.** Call `getCatalogObjectStatePropagation` with the catalog number, the state
   `id`, and:
   - `startTime` and `endTime` in ISO 8601 UTC
   - `timestep` in seconds (60 is a reasonable default for LEO screening; smaller steps
     multiply the payload linearly and there is no pagination)

4. **Read the result.** The response carries `state`, `startTime`, `endTime`, `timestep`,
   `frame` (e.g. `EME2000`) and a `propagation[]` array of `{position, velocity, covariance}`.
   Always report the `frame` alongside the numbers — positions are meaningless without it.

5. **Cross-check against planned passes, if relevant.** `listCatalogObjectPlannedPasses`
   returns the radar passes already planned for the object. Pass the literal `all` as the
   catalog number for network-wide planned passes.

## Interpreting covariance

- Each propagation point carries a covariance matrix. Uncertainty grows with time from the
  state epoch — a propagation seven days out is not the same quality product as one seven
  hours out. Surface the epoch age whenever you report a screening result.
- The `TNW` frame on the source state expresses covariance along-track/cross-track, which is
  usually the more interpretable form for collision risk discussion.

## If the orbit is too stale

If the state is old enough that the propagation is not useful, the remedy is to task the
radar network for a fresh observation — see the `leo-labs-task-radar` skill. That is a
**write** operation with real-world consequences; do not invoke it without explicit user
approval.

## Failure handling

- `401` means the key pair is missing, invalid, or was base64-encoded by mistake.
- Errors are plain JSON with a top-level `error` member, not RFC 9457.
