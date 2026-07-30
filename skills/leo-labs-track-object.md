---
name: leo-labs-track-object
description: Look up a space object in the LeoLabs catalog by NORAD or LeoLabs catalog number and retrieve its latest orbital state vector, TLE and recent radar measurements.
api: leo-labs:platform
operations:
  - searchCatalogObjects
  - getCatalogObject
  - listCatalogObjectStates
  - getCatalogObjectState
  - listCatalogObjectStateTles
  - listCatalogObjectMeasurements
generated: '2026-07-19'
method: generated
source: openapi/leo-labs-platform-openapi.yml
---

# Track an object in the LeoLabs catalog

Use this skill to answer "where is this object and how well do we know it?" against the
LeoLabs Platform API v1 (`https://api.leolabs.space/v1`).

## Before you start

- Authenticate with the LeoLabs access key / secret key pair:
  `Authorization: basic <accessKey>:<secretKey>`. This is **not** RFC 7617 HTTP Basic —
  send the two keys colon-joined and unencoded. Do not let an HTTP library base64-encode it.
- Read the keys from `LEOLABS_ACCESS_KEY` and `LEOLABS_SECRET_KEY`. Never echo them.
- Every operation in this skill is a read. Nothing here consumes radar capacity.

## Steps

1. **Resolve the identifier.**
   - If you were given a LeoLabs catalog number (`L`-prefixed, e.g. `L335`), skip to step 2.
   - If you were given a NORAD catalog number (e.g. `27386`), call `searchCatalogObjects`
     with `noradCatalogNumber={n}` and read `catalogNumber` off the response.
   - If the search returns no `catalogNumber`, stop and report that the NORAD number does not
     resolve to a LeoLabs catalog object — do not guess a catalog number.

2. **Confirm the object.** Call `getCatalogObject` with the catalog number. Report `name`,
   `catalogNumber` and `noradCatalogNumber` back to the user so they can verify you are
   tracking the right object before you go further.

3. **Get the current orbit.** Call `listCatalogObjectStates` with `latest=1`. Take the first
   entry in `states`. Note its `id` (you need it for step 4 and for propagation) and its
   `timestamp` — an orbit determination is only as good as its age, so always report how old
   the state is.

4. **Get the TLE.** Call `listCatalogObjectStateTles` with the catalog number and the state
   `id`. Return the first entry of `tles`. (`getCatalogObjectState` in the official client
   performs this same join and merges the result in as `tle`.)

5. **Assess measurement support, if asked.** Call `listCatalogObjectMeasurements` with
   `startTime` and `endTime` in ISO 8601 UTC. Both are required — there is no default window,
   and no pagination, so keep the window tight (hours, not weeks) or you will pull a very
   large payload.

## Reading the data

- State vectors are published per reference frame under `frames`. Use `EME2000` for inertial
  work; `TNW` is the local orbit frame useful for expressing covariance along-track.
- Measurements carry both `values` (raw observables) and `corrected` (after bias and
  ionospheric corrections), with a `corrections[]` audit trail. **Use `corrected` for
  analysis** unless you specifically need the raw radar return.
- Units are SI throughout: metres, metres per second, seconds, hertz, watts, degrees.

## Failure handling

- A `401` means the key pair is missing or invalid, or was base64-encoded by mistake.
- Errors come back as plain JSON with a top-level `error` member — not RFC 9457 problem
  details. Do not expect a `type` or `title`.
- Reads are safe to retry. The official client retries any non-2xx/3xx up to 5 times.
