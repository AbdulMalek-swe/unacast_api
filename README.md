# Unacast API — trial reference and response log

**Owner:** Malek · **Repo path:** `docs/unacast/API_REFERENCE.md`

This is the working document for the Unacast trial. Hit one endpoint at a time,
paste the real response in, answer the questions under it. By the end of the trial
this file is the response contract we never had, and everything downstream gets
easier.

Keep it in the repo, not in Google Docs. Responses are large JSON — they need code
blocks, diffs and history.

---

## Progress

| # | Endpoint | Priority | Status |
|---|---|---|---|
| 1 | `POST /areas/devices` | **First** | ⬜ |
| 3 | `POST /observations/geo/search` | **First** | ⬜ |
| 8 | `GET /requestStatus/{requestID}` | **First** | ⬜ |
| 2 | `POST /areas/devices/trends` | Second | ⬜ |
| 5 | `POST /areas/tradeareas` | Second | ⬜ |
| 4 | `POST /observations/registrationID/search` | Third | ⬜ |
| 6 | `GET /personas/taxonomy` | Third | ⬜ |
| 7 | `POST /areas/personas` | Third | ⬜ |

⬜ not called · ✅ response captured · ⚠️ something unexpected

**Why that order.** Endpoints 1 and 3 are the whole product — 1 is what the user
sees while browsing, 3 is the nightly sweep everything else derives from. Endpoint
8 is small but it gates the biggest open cost question, so find out early. The rest
can wait.

---

## Start here — do these before the first call

1. **Put the key in Secret Manager.** It arrives on a Bitwarden Send link. Those
   expire and can self-destruct on first open, so whoever opens it places it
   immediately. Not in `.env`, not in Slack, not in this file, not in a commit.

2. **Turn on response capture before anything else.** See
   [Stop hand-pasting responses](#stop-hand-pasting-responses). This is the single
   highest-value thing in the trial and it takes twenty minutes.

3. **Use staging for first calls.**
   `https://api.gravyanalytics.com/v1.1-staging/`
   Ask Unacast whether staging counts against the monthly cap before running
   volume through it.

4. **Check the three boot assertions on the first live response:**
   - the key returns `advertiserID`, **not** `registrationID`
   - country scope is USA and CAN
   - export is enabled on the key

   A contract term and a key configuration are different things. The gap between
   them shows up in production unless somebody checks on day one.

### Auth

```http
Authorization: <API_KEY>
Content-Type: application/json
Accept: application/json
Accept-Encoding: gzip
```

The **raw key**. No `Bearer`, no scheme prefix, no quotes. This is the detail most
integrations get wrong on the first call, and a wrong header gives you a 401 with
no useful message.

### Base URLs

| | |
|---|---|
| Production | `https://api.gravyanalytics.com/v1.1/` |
| Staging | `https://api.gravyanalytics.com/v1.1-staging/` |

---

## The envelope — worth memorising

| Limit | Value |
|---|---|
| Concurrent direct queries | **8** |
| Concurrent batch exports | **2** |
| Monthly request cap | **100,000** |
| What counts as a request | **A successful query that returns a response.** Errors do **not** count |
| Rollover | **None.** Unused capacity expires each month |
| Countries | **USA and CAN** |
| Advertiser IDs in delivery | Yes |
| Processing timeout | **180 s**, then 408 |
| Hosting | AWS us-east (Virginia) |

Eight slots are shared across every user on the platform. **Every call goes
through the broker** — not a script, not a notebook, not a debug session.

---

## ⚠️ Correction — countries

Order 002 reads: *"Custom list of countries: **USA & CAN**"*.

**Mexico is not in scope.** The public Unacast docs mention Mexico because their
platform supports it, but our licence does not. If we build against Mexico:

- the country guard won't reject it,
- a user builds an audience we can't deliver,
- and we've queried data we aren't licensed for.

Reject MEX at the first screen, before any request is made.

---

## Five rules

These are the ones that produce results which look completely reasonable and are
wrong. Each has a regression test that stays forever.

1. **Never sum `deviceCount` across features.** The de-duplicated total is
   `crs.properties.uniqueDeviceCount`. A device that visited two places is one
   device. Summing double-counts and the number looks plausible.

2. **`inMin` is not group-AND.** It means "any N of the features *in this
   request*". For "vet AND PetSmart AND dog park" with `inMin: 3` you get people
   who visited three vets — the opposite of a qualified pet owner. Group-AND is a
   client-side sketch intersection. `inMin` is right for exactly one thing: N-of-M
   inside a **single** group, like "3+ of my own stores".

3. **Never retry a 408 identically.** It means the request was too big. An
   identical retry times out again and holds one of eight slots for another 180
   seconds — somebody else pays that in latency.

4. **Never stitch two 90-day windows and add the unique counts.** That
   double-counts every returning device.

5. **Trade Areas: `decisionLocationTypes` is `["ZIPCODE", "CBG"]`, hard-coded.**
   Never LATLNG, never GEOHASH. The API default is all five, so **omitting the
   field is the mistake**. This is a constant in code, not a config value.

---

## Status codes — fill in what we actually see

| Code | Meaning | What the client does | Seen? |
|---|---|---|---|
| `2xx` | Success | Return body. **Only this counts against the cap** | ⬜ |
| `400` | Bad request, our bug | **Never retry.** Validator should have caught it | ⬜ |
| `401` | Key, header or scope problem | Do not retry. Page someone | ⬜ |
| `406` | Feature or area cap exceeded | Do not retry. Split and resubmit | ⬜ |
| `408` | Exceeded 180 s | **Never retry identically.** Shrink | ⬜ |
| `410` | Export result expired | Resubmit the export | ⬜ |
| `429` | Rate limited | Circuit break. Drain with jitter | ⬜ |
| `5xx` | Their side | Retry with jitter, bounded | ⬜ |

**Capture the body of every error, not just the code.** Error bodies are where the
detail is, and they're the hardest thing to reproduce later on demand.

---

# 1. Area Devices ⬜

```http
POST /v1.1/areas/devices
```

Device and visitor counts for an area. This is the **browse call** — behind the
per-pin numbers and the total the user sees.

| | |
|---|---|
| Geometry | Point, LineString, Polygon |
| Max features | **20** per request |
| Max area per feature | **Not stated in the docs.** We assume 400 km² — probe it and record below |
| Max total area | **Not stated** for this endpoint |
| Direct window | 90 days |
| Countries | USA, CAN |

### Fields that matter

| Field | Why |
|---|---|
| `deviceCountOnly` | `true` returns counts with no identifiers. **This is the entire cost model** — browsing must never pull identifiers |
| `excludeFlags` | Forensic mask. Default BALANCED = `411067008` |
| `placeIdentifier` | `["NONE"]` when browsing. Enrichments add processing time |
| `responseType` | `DIRECT` or `EXPORT` |
| `inMin` | See rule 2 |

### Reading the response

| Read this | Not this |
|---|---|
| `crs.properties.uniqueDeviceCount` — de-duplicated total | ❌ `features[i].properties.deviceCount` summed. See rule 1 |
| `features[i].properties.deviceCount` — per-pin label only | |
| `deviceLimitHit` — vendor truncated that feature at 10k | |

### Request

```json
{
  "type": "FeatureCollection",
  "crs": { "type": "name", "properties": {
      "inMin": 1,
      "placeIdentifier": ["NONE"],
      "responseType": "DIRECT" }},
  "features": [{
    "type": "Feature", "id": "<our place_id>",
    "properties": {
      "radiusInMeters": 100,
      "startDateTimeEpochMS": 0, "endDateTimeEpochMS": 0,
      "excludeFlags": 411067008,
      "deviceCountOnly": true },
    "geometry": { "type": "Point", "coordinates": [0, 0] }
  }]
}
```

### Response

```json
PASTE REAL RESPONSE HERE
```

### Answer these

- Exact path to the de-duplicated total — is it really `crs.properties.uniqueDeviceCount`?
- Area cap probe: submit a deliberately oversized polygon. What's the actual limit?
- Response time for 1 feature / 20 features?
- What does `deviceLimitHit` look like when it fires?
- Anything surprising:

---

# 2. Area Device Trends ⬜

```http
POST /v1.1/areas/devices/trends
```

Daily, weekly and monthly breakdowns for an area.

| | |
|---|---|
| Geometry | Point, LineString, Polygon |
| Max features | **20** |
| Max area per feature | **400 km²** |
| Max total area | **1,000 km²** |
| Direct window | 90 days |
| Countries | USA, CAN |

⚠️ Don't write "same body as endpoint 1". The request is similar; the **response
shape is different**, and that's the half we need recorded.

### Request

```json
PASTE THE REQUEST YOU ACTUALLY SENT
```

### Response

```json
PASTE REAL RESPONSE HERE
```

### Answer these

- How are the daily / weekly / monthly buckets keyed?
- Bucket boundaries — UTC or local?
- Does the trend total match `uniqueDeviceCount` from endpoint 1 for the same area and window? If not, why not?

---

# 3. Geographic Observations ⬜

```http
POST /v1.1/observations/geo/search
```

Raw observations inside an area. This is the **sweep call** — one pull per place
per day, from which distance bands, visits, dwell and hour bits are all derived.
The most important endpoint in the integration.

| | |
|---|---|
| Geometry | Point, LineString, Polygon, MultiPolygon |
| Max features | **10** — not 20. Different from every other endpoint |
| Max area per feature | **25 km²** — not 400 |
| Point / LineString | `radiusInMeters` required |
| Direct window | 90 days |
| Export window | **Full 3-year rolling range** |
| Max observations | 100,000 per search object |
| Countries | USA, CAN |

The 10-feature and 25 km² caps are the two most commonly hit limits in the whole
integration. Put them in the validator before the first call.

### Sweep strategy

Pull once at the **widest radius** — 500 m, the outer band edge. Every narrower
radius is derived from the same observations. Pulling per-radius multiplies the
request count by ten and buys nothing.

### Request

```json
{
  "type": "FeatureCollection",
  "crs": { "type": "name", "properties": {
      "inMin": 1,
      "observationLocationTypes": ["LATLNG"],
      "responseType": "DIRECT" }},
  "features": [{
    "type": "Feature", "id": "<our place_id>",
    "properties": {
      "radiusInMeters": 500,
      "startDateTimeEpochMS": 0, "endDateTimeEpochMS": 0,
      "excludeFlags": 411067008,
      "returnObservations": true },
    "geometry": { "type": "Point", "coordinates": [0, 0] }
  }]
}
```

### Response

```json
PASTE REAL RESPONSE HERE — at least 3 observations so we can see the shape
```

### Answer these — this list matters most

- **Exact field names on one observation.** lat, lng, timestamp, registrationID, flags — what are they actually called?
- **Is the timestamp epoch milliseconds, and is it UTC?**
- Are forensic flags returned per observation, or only used to filter?
- How many observations come back for a busy place on one day?
- **Pagination — how does it work? Is there a cursor?**
- Does the response tell us when it truncated?

---

# 4. Device Observations by Registration ID ⬜

```http
POST /v1.1/observations/registrationID/search
```

Observations for known registration IDs. Used after endpoint 3, once IDs are known.

| | |
|---|---|
| Input | `registrationIDs`, each a valid UUIDv4 |
| Geometry | Not required |
| `inMin` | Not required |
| Max IDs | **100** direct · **1,000** export |
| Window | **Rolling 3 years** — not 90 days |
| Default start if omitted | **3 years ago** — not 7 days like the others |

⚠️ Two things differ from every other endpoint: the 3-year window and the default
start date. Both are easy to get wrong by assuming consistency.

### Request

```json
PASTE THE REQUEST YOU ACTUALLY SENT
```

### Response

```json
PASTE REAL RESPONSE HERE
```

### Answer these

- Is the response shape the same as endpoint 3, or different?
- What happens with an ID that has no observations — omitted, or empty entry?
- What happens with a malformed UUID — whole request 400, or just that ID skipped?

---

# 5. Trade Areas ⬜

```http
POST /v1.1/areas/tradeareas
```

Where a place's visitors come from. The highest-value B2B output we have.

| | |
|---|---|
| Max features | 20 |
| Max area per feature | 400 km² |
| Max total area | 1,000 km² |
| **Privacy floor** | **50 unique pseudonymised registration IDs per feature.** Below that the feature returns **nothing** — not an error, a 200 with an empty result |
| `maxDecisions` | Default 1 |
| `decisionLocationTypes` | Defaults to **all five**: LATLNG, GEOHASH, ZIPCODE, COUNTY, CBG |

### 🚫 Hard rule — see rule 5

```json
"decisionLocationTypes": ["ZIPCODE", "CBG"]
```

Never LATLNG. Never GEOHASH. The default includes them, so **omitting the field is
the mistake**. Requesting origin data at coordinate precision means inferring where
people live. It is the most expensive possible error in this integration.

Because of the 50-device floor, call this at **chain-market level**, not per store.
A single store usually won't clear the floor and will silently return nothing.

### Request

```json
{
  "crs": { "properties": {
      "decisionLocationTypes": ["ZIPCODE", "CBG"],
      "includeAdditionalZipInfo": true,
      "maxDecisions": 1,
      "responseType": "DIRECT" }}
}
```

### Response

```json
PASTE REAL RESPONSE HERE
```

### Answer these

- **Confirm a sub-50-device feature returns 200 with an empty result, not an error.** Test this deliberately with a small store.
- What does `includeAdditionalZipInfo` actually add?
- With `maxDecisions: 1`, what exactly is the one decision?

---

# 6. Personas Taxonomy ⬜

```http
GET /v1.1/personas/taxonomy
```

The list of available persona segments. Call once, cache 24 h.

### Response

```json
PASTE REAL RESPONSE HERE — the full taxonomy
```

### Answer these

- How many segments are there?
- **Are any of them health, financial, political, religious or otherwise off-limits under our own blocklist?** Flag those to Roman before anyone builds UI around the taxonomy.

---

# 7. Area Personas ⬜

```http
POST /v1.1/areas/personas
```

Persona composition of visitors to an area.

| | |
|---|---|
| Max features | 20 |
| Max area per feature | 400 km² |
| Max total area | 1,000 km² |
| Max IDs | 100,000 |

⚠️ Personas describe **areas, not people**. Never present a persona result as a
statement about an individual, and never use one to infer a protected attribute.

### Request

```json
PASTE THE REQUEST YOU ACTUALLY SENT
```

### Response

```json
PASTE REAL RESPONSE HERE
```

### Answer these

- Are results percentages, counts, or an index?
- Is there a minimum-size floor like Trade Areas has?

---

# 8. Request Status ⬜

```http
GET /v1.1/requestStatus/{requestID}
```

Poll for the state of an EXPORT request. Referenced by the API docs; we never
received separate documentation for it.

### 🔴 The most important open question in the integration

Unacast defines a request as *"a successful query that returns a response"*.
A `requestStatus` poll returns 200. **On that definition it counts.**

If it counts:

- a 5-second poll on a 10-minute export is **120 counted requests**
- 500 exports a month polled that way is **60,000 requests**
- that's most of our 100,000 monthly cap, spent asking "are you done"

So the poller uses a **backoff schedule, not an interval**:

```
30 s → 60 s → 120 s → 300 s → 300 s … capped at 1 hour total
```

That takes a ten-minute export from ~120 polls to about **6**.

**Confirm this with Unacast before the deep backfill runs.** The backfill is
exactly the workload that would find the answer the expensive way.

### Response

```json
PASTE REAL RESPONSE HERE — both an in-progress state and a completed state
```

### Answer these

- What states exist?
- Does the completed response contain the download URL directly?
- How long is the result available before 410?
- **Confirmed whether polls count against the cap?**  ⬜ asked  ⬜ answered

---

## Stop hand-pasting responses

Filling this in by hand works for the first few calls and then stops happening.
Make the response half a by-product of running the code instead:

```python
# app/services/unacast/client.py

if settings.UNACAST_CAPTURE_RESPONSES:
    _capture(path, body, r)


def _capture(path, body, response):
    ts = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%S%f")
    slug = path.strip("/").replace("/", "_")
    out = Path(settings.UNACAST_CAPTURE_DIR) / f"{ts}_{slug}_{response.status_code}.json"
    out.parent.mkdir(parents=True, exist_ok=True)
    out.write_text(json.dumps({
        "request":  {"path": path, "body": body},
        "response": {"status": response.status_code,
                     "headers": dict(response.headers),
                     "body": _safe_json(response),
                     "elapsed_ms": int(response.elapsed.total_seconds() * 1000)},
    }, indent=2))
```

```python
# app/core/config.py
UNACAST_CAPTURE_RESPONSES: bool = False
UNACAST_CAPTURE_DIR: str = "/tmp/unacast_capture"
```

Set it `true` for the whole trial. Every call writes itself to disk, errors
included. Those files become three things at once:

1. **The response contract** — the half this document is missing
2. **Test fixtures** — real data, not invented data
3. **A mock server** — so the rest of the team builds without burning trial
   requests or waiting on a credential

Trial credentials against unfamiliar endpoints are a window that doesn't reopen.
Everything after this is easier if the data exists and materially harder if it
doesn't.

---

## Ask Unacast

Send as one numbered email. Tick when answered.

- [ ] **Does a `requestStatus` poll count against the monthly cap?** ← ask first
- [ ] Do staging calls count against the cap?
- [ ] Is there a daily rate limit underneath the monthly cap? The v1.1 docs mention a per-key daily limit returning 429 at 00:00 UTC
- [ ] Is the month calendar-based, or rolling from the Order Effective Date?
- [ ] Is the 8/2 concurrency per key or per account?
- [ ] Is the area cap on `/areas/devices` the same 400 km² the other endpoints state? The docs are silent
- [ ] Is the 90-day window lifted for EXPORT on Visitors, Personas and Trade Areas, as it is for Observations?
- [ ] The `requestStatus/{requestID}` documentation — referenced but never received
- [ ] Written acknowledgement of our static egress IP `34.19.228.197`
- [ ] How recent is the freshest observation available?

---

## Done when

- [ ] Response capture on before the first call
- [ ] All 8 endpoints called, real responses pasted in
- [ ] Every question under every endpoint answered
- [ ] Error responses captured too — especially 400, 406, 408
- [ ] Mexico removed everywhere; country guard rejects MEX at the first screen
- [ ] Validator enforces 10 features / 25 km² on observations, 20 / 400 elsewhere
- [ ] `decisionLocationTypes` hard-coded to ZIPCODE + CBG
- [ ] Regression tests exist for rule 1 (never sum `deviceCount`) and rule 2 (`inMin` is not group-AND)
- [ ] Export poller uses backoff, not a fixed interval
- [ ] Every call goes through the broker
- [ ] The Unacast question list sent

---

## Send back to Roman at the end

1. This file, filled in
2. The capture directory, zipped
3. A short note on anything that contradicts this document — **those are the most
   valuable findings and they should not sit in your head**
