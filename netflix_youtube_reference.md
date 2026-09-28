# Video Streaming (Netflix/YouTube) — Reference (Full Mock)

Study reference (~60% — your lowest, but because it's the most UNFAMILIAR domain, not a
reasoning failure). The gap was not knowing the domain pipeline (transcode + chunk + ABR).
Pure knowledge — closed by reading this. Key insight you DID get: metadata ≠ content.

**Architecture diagram:** see `netflix_youtube_architecture.svg`.

---

## 1. Requirements

**Functional**
- Creators upload videos (any size)
- Viewers watch on any device, quality adapts to their network
- Watch from any region (global delivery)
- Resume playback across devices
- Subscriptions/premium out of v1

**Non-functional**
- **THE defining number is VIDEO storage — petabytes, not terabytes:**
  10M videos/day, each ~GBs raw, stored in ~5-7 resolutions → **~50 PB/day** of content.
  (Metadata is ~20 TB — a rounding error next to the video bytes.)
- **Load is VIEW-bandwidth, not uploads:** uploads ~115/s (low); views = billions/day,
  millions of CONCURRENT streams, petabytes/day of egress. That's the crushing load.
- Low startup latency (time-to-first-frame), minimal rebuffering
- High availability, global

---

## 2. High-Level Design — the pipeline IS the problem

**This is a static content-delivery problem, not a transaction/DB problem.** Two halves:

### WRITE path — upload & encode (the part most people skip)
```
1. Creator uploads raw video → directly to S3 (via pre-signed URL)
   — NOT through app servers (a 5GB upload can't tie up a request thread)
2. Upload triggers the ENCODING PIPELINE (async, via a queue):
   Encoding workers transcode the raw file into:
     - MULTIPLE RESOLUTIONS: 240p, 480p, 720p, 1080p, 4K
     - CHUNKED: each resolution split into ~2-10s segments
     - a MANIFEST file (HLS/DASH) listing every chunk at every resolution
3. Encoded chunks → S3 → distributed to CDN edges
```
The transcode-into-chunked-multi-resolution is what makes the video streamable AND
adaptive. Without it there is no streaming. This is the heart of the write path.

### READ path — playback (CDN is 95% of the design)
```
4. Viewer hits play → player fetches the MANIFEST from the nearest CDN edge
5. Player streams the video CHUNK BY CHUNK from the CDN edge near them
   (Tokyo viewer → Tokyo edge, NOT your Virginia origin — the CDN absorbs the bandwidth)
   On a cache miss, the edge pulls from origin (S3) once, then serves cached.
6. ADAPTIVE BITRATE: for each chunk, the player measures current bandwidth and picks the
   resolution — good wifi → 1080p chunk, wifi drops → 480p chunk — seamlessly mid-video,
   because every chunk exists at every resolution (created in step 2).
7. Playback position reported periodically → Kafka → DB → resume across devices.
```

### Metadata — a SEPARATE system
Title, description, view count, comments, recommendations → a normal DB/service.
Completely separate from the video bytes (which live in S3 + CDN). Don't conflate them.

---

## 3. Schema (LLD) — metadata only (content is in S3/CDN, not the DB)

```
videos
  id PK, creator_id FK, title, description,
  manifest_url  (points to the HLS/DASH manifest in S3/CDN),
  status (PROCESSING | READY | FAILED),
  created_at
  → index (creator_id, created_at DESC)   "a creator's videos, recent first"
  → index (status)                         "find videos still processing / failed"

watch_progress
  id PK, user_id FK, video_id FK, position_seconds, updated_at
  → index (user_id, video_id)              "resume: last position for this user+video"
  → unique (user_id, video_id)

(view counts / comments → their own tables/services; recommendations → separate system)
```

Note: `manifest_url` is just a reference — the actual video chunks are in S3/CDN, never
in the DB (same pattern as file_url in chat/news feed).

---

## 4. Failure handling
- **Encoding job fails** → retry from the queue; video stays `PROCESSING` (not publishable
  until all resolutions are encoded + pushed to CDN). Retry granularly — a failed single
  resolution/chunk shouldn't re-encode everything (idempotent, per-chunk retry).
- **CDN edge down** → fall back to another edge / origin (S3).
- **Upload interrupted** → resumable/chunked upload (don't restart a 5GB upload from zero).

## 5. Observability
- **Monitor:** rebuffering rate (THE user-experience metric — playback stalls),
  time-to-first-frame (startup latency), **CDN cache hit ratio** (health of the whole
  delivery model — low = edges constantly hitting origin), encoding queue backlog.
- **Page on:** rebuffering rate spikes, cache hit ratio drops, encoding backlog grows.

---

## Where YOUR attempt differed (fixes)
1. **Skipped the encoding pipeline** — you went upload → CDN directly. You cannot serve the
   raw file; it MUST be transcoded into chunked multi-resolution first. That step is the
   heart of the write path and the thing interviewers most want to hear.
2. **Skipped adaptive bitrate** — you noted "quality by network" in requirements but didn't
   design it. It works BECAUSE of the chunked multi-resolution: player picks resolution
   per-chunk by measured bandwidth.
3. **Missed the storage magnitude** — video is PETABYTES/day (raw × 5-7 resolutions), the
   defining number. You computed only metadata (20 TB).
4. **Load is view-bandwidth, not upload QPS** — millions of concurrent streams, PB/day egress.
5. **Got right:** metadata/content separation, CDN distribution, resume-watching via Kafka.

**One-line takeaway:** the gap here was DOMAIN KNOWLEDGE of the video pipeline (transcode →
chunk → manifest → CDN → adaptive bitrate), not reasoning. Read this once and it's closed.
The spine to memorize: upload → S3 → encode to resolutions+chunks+manifest → CDN edges →
player streams chunks, adapting quality per chunk. CDN + chunked-multi-resolution encoding
are the whole game.
