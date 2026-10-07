<!-- day-nav -->
[← Day 57 — Product search](57-product-search.md) · [Day 59 — Live video →](59-live-video.md)

# Day 58 — Video on demand

**Do now**

1. Set a timer.
2. Attempt the problem. Stop at the attempt line. Do not scroll.
3. Then read.

## Time box

40 minutes.

| Minutes | Do this |
|---|---|
| 0–12 | Attempt. Upload through transcode to CDN playback. Bytes off the app tier. |
| 12–32 | Read. If the app streams 4 GB files through itself, redo storage. |
| 32–40 | Say 201 vs ready-to-play, and where ABR renditions live. |

## Intent

Facing a long video, leave able to take upload through transcode to CDN playback without bytes sitting on the app tier.

## Problem

> Design video on demand.
>
> People upload a video. Viewers play it with adaptive bitrate. Playback should come from an edge, not from your application servers.

## Attempt before reading

12 minutes. Do not scroll. Day 28 image upload is the cousin; duration and ABR are new.

Write:

1. Upload: direct-to-object-store or through app?
2. When is upload "done"? When is video "playable"?
3. Transcode graph: renditions, packaging (HLS/DASH).
4. Playback path: manifest + segments on CDN.
5. Hot video egress math.

---

**Stop. Direct upload. Async transcode. CDN for manifests and segments.**

---

## Requirements

**In.** Upload up to **10 GB**, **≤ 3 hours**. Transcode to a ladder (e.g. 360p/720p/1080p). HLS or DASH. Playback via CDN. Owner delete. Status: processing | ready | failed.

**Assumptions.** **100,000** uploads/day ≈ **1.2**/s; mean size **500 MB** → **50 TB/day** ingest. Views: **50** per video life mean; hot titles dominate egress. One region origin; CDN global.

**Out.** Live (day 59), recommendations, comments, DRM deep dive (mention Widevine/FairPlay as dependency if asked), mezzanine editing suite.

## Estimates

Ingest 50 TB/day. Transcode CPU: rough **0.5–2× realtime** per rendition — pool sized for backlog, not upload ack. Egress: one viral 4 Mbps stream × 100k concurrent = **400 Gbit/s** — CDN or death.

App NIC must not see segment traffic.

## API and data

`POST /v1/videos` → upload instructions (signed PUT URLs / multipart).

`POST /v1/videos/{id}/complete` → validate object, enqueue transcode, 202.

`GET /v1/videos/{id}` metadata + `playready` + manifest URL when ready.

`GET` manifest/segments via CDN (tokenized).

Row: id, owner, state, duration, renditions[], error.

Objects: `vid/{id}/original`, `vid/{id}/hls/...`.

## Design

### Upload

Signed multipart to private bucket. App never buffers the 10 GB. Complete: head object, size check, virus scan optional async, commit row `processing`, outbox job.

### Transcode

Workers pull original, produce renditions + playlist, write segments, set `ready`. Idempotent by derived keys. Poison file → `failed`, keep original for debug.

### Playback

CDN in front of segment bucket (or packager). Manifest short TTL; segments long immutable cache. Token auth on manifest if private content.

### Delete

Row delete + detach job for objects; CDN purge best-effort; segment immutability means old URLs may work until TTL/purge — tokens should expire.


### Staff arithmetic: ingest, backlog, egress

50 TB/day ingest is ~4.6 Gbit/s average into the bucket — the app never sees it if signed PUT works. Transcode at 1× realtime for three renditions on a 3-hour mezzanine is ~9 hours of CPU per video if serial; parallelize renditions and size the worker pool from **backlog age**, not from upload QPS. A 202 that lies "ready" because the original landed is the failure mode you refuse.

Viral egress: 100k concurrent × 4 Mbps = **400 Gbit/s**. That number is why segments are CDN objects with long cache on immutable paths. Manifest short TTL (e.g. 2–5 s during active publish of new playlists; longer once static) so a new rendition appears without waiting on year-long CDN cache.

**Poison and idempotency.** Derived keys `vid/{id}/hls/720p/...` make retries safe. A bad original marks `failed` and keeps the mezzanine for debug — do not delete the evidence on first worker exception.



### Worked path: upload → first playable

1. `POST /videos` → signed multipart URLs; client PUTs to bucket.
2. `complete` → head object, size/type checks, row `processing`, outbox.
3. Worker writes `360p` segments first → flip `ready` with only that rung; continue 720p/1080p.
4. Player fetches CDN manifest; missing higher rungs simply omit from playlist until present.

**Owner delete while encoding.** Cancel jobs; detach objects; row gone. In-flight workers must no-op if row missing (idempotent derived keys help).

## Diagrams

```mermaid
sequenceDiagram
  participant C as Client
  participant A as App
  participant B as Bucket
  participant W as Transcode
  participant CDN as CDN
  C->>A: create + signed URLs
  C->>B: multipart PUT
  C->>A: complete
  A-->>C: 202 processing
  W->>B: read original write HLS
  C->>CDN: GET manifest/segments
```

Caption: "App authorizes. Bytes skip the app. Play from edge."

## Failure the user sees

**Transcode backlog.** Upload complete; spinner on play until ready. Page on oldest processing age and queue depth. Original is not playable unless you explicitly offer a progressive mezzanine (usually no for consumer ABR).

**CDN origin miss storm on premiere.** Prefetch popular titles; origin shield. App still not in path. User-visible: buffering, not a 500 from your API.

**Bucket down on upload.** Complete fails; no `processing` state. Partial multipart abandoned by lifecycle rules — say the abandon TTL.

**Ready with only 360p.** User can play early; 1080p appears later. If you wait for the full ladder before `ready`, premiere delay grows with CPU — say which you pick (prefer early play).

**Delete while viewers watch.** Tokens expire; CDN may still serve immutable segments until TTL/purge. Do not promise instant global blackout; promise row gone + new manifests 404.

## Trade-offs

**Client-side vs server-side packaging.** Server ladder controls quality; client upload of all renditions burns user bandwidth — refuse for consumer VOD.

**Ready before all renditions.** Allow play with subset (360p first) — better UX; say it.

**Name the refusal inside each alternative.** Against app-proxied upload: you refuse melting the NIC on 10 GB multiples. Against 201-ready on complete: you refuse a play button that 404s manifests. Against app-served segments: you refuse 400 Gbit/s through your tier. Against client-encoded ladders only: you refuse quality and CPU on the uploader's phone as the product. Against instant global delete: you refuse a promise immutable CDN caches cannot keep.

**10× uploads.** ~12/s and 500 TB/day — bucket and transcode pool, same topology. Egress 10× is still CDN.

## Talking points

**If bytes through app.** "10 GB × concurrent uploads melts us. Signed PUT."

**If they ask when play works.** "State ready after first playable rung, not on upload complete. 202 means original durable and job queued."

**If they ask about DRM.** "Widevine/FairPlay as a dependency on the packager; license server is out of today's deep dive unless they insist."

**If they ask what pages.** "Transcode backlog age. Upload error rate. CDN origin bandwidth — not app 5xx on play."

**If they ask about a failed transcode.** "State failed; keep original; do not flip ready. Owner can retry; derived keys stay idempotent."

## Say this in the room

Uploads go direct to object storage with signed multipart; complete enqueues transcode and returns 202 until HLS renditions exist — I prefer marking ready when the first ladder rung is playable, not when 1080p finishes. Playback is CDN-fronted manifests and immutable segments; a viral title at 100k concurrent and 4 Mbps is about 400 Gbit/s, an edge problem, never an app NIC problem. Transcode is idempotent on derived keys; failures mark failed without lying that the video is ready. Delete tombs the row and detaches objects asynchronously; tokens expire because purge is best-effort.

### Staff depth: ready vs complete, and viral egress

202 on complete means original durable and job queued — not playable. First ladder rung may unlock play before 1080p finishes — say it.

Viral: 100k concurrent × 4 Mbps ≈ **400 Gbit/s** — CDN or death. App never serves segments.

**Delete vs immutable segments.** Tokens expire; purge is best-effort; do not promise instant global unavailability.

**Backlog sizing.** Size workers from oldest-processing age, not from upload QPS. Upload can be fine while play is a desert.

**What staff sounds like.** Direct upload, async ladder, edge playback, idempotent derived keys, 400 Gbit/s said before the brand of CDN.

### More probes, with the answer

**"What pages?"** Name the user-visible lag or error metric from Failure — not only CPU. **"What do you refuse?"** Pick one refusal from Trade-offs and say the lie it prevents. **"What is the sensitive assumption?"** The estimate that flips the design if wrong by 10×.

## Kit artifact

| Follow-up | You answer with |
|---|---|
| "When can I play?" | State ready after first playable ladder; not on upload complete. |

## Design log

One line: whether upload bytes touched the app in your attempt, and the ready vs complete distinction.

Next: [Day 59 — Live video](59-live-video.md). Ingest, ABR, start stampede.

---

<!-- day-nav -->
[← Day 57 — Product search](57-product-search.md) · [Day 59 — Live video →](59-live-video.md)
