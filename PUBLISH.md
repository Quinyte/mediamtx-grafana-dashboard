# Publish drafts

Copy for the three places this dashboard needs to be announced. **Nothing here is
published — a human posts all of it.**

Before posting, replace every `<REPO_URL>` with the real repository URL. The README
still carries the `github.com/<you>/` placeholder and the repo has no git remote yet,
so the URL does not exist at the time of writing.

`<GRAFANA_ID>` is assigned by grafana.com when the dashboard is first published. The
issue replies below link the repo, not the registry, so they can go out either before
or after the registry listing.

**Blocker on draft 3:** mediamtx issue **#163 is closed and locked** (auto-locked
2023-01-01 after 6 months closed). Comments are disabled — nobody can post to it, not
even a maintainer, without unlocking. Draft 3 is written as asked, plus the routing
options for actually reaching that thread's audience. See the note above it.

---

## 1. grafana.com dashboard registry listing

### Name

```
MediaMTX — Fleet Overview
```

### Summary (one line)

```
Fleet-wide view of MediaMTX servers from the native Prometheus endpoint — paths, sessions by protocol, bitrate, and RTP loss/jitter across every node.
```

### Description

```markdown
A Grafana dashboard for [MediaMTX](https://github.com/bluenviron/mediamtx) built
directly on its native Prometheus metrics endpoint. No exporter, no sidecar, no
config beyond `metrics: yes`.

**It aggregates across nodes, not just one.** MediaMTX exports *per-entity*
metrics — every path, session, and connection is its own labeled time series, and
a raw `paths` query on a fleet returns thousands of series that mean nothing on a
graph. This dashboard does the aggregation: `count()` over the entity gauges for
live totals, `rate(bytes) * 8` for real bitrate, `sum by (state)` and
`sum by (readerType)` for the breakdowns. Point one Prometheus at fifty MediaMTX
servers and the Overview row is the fleet; scope it back down with the `path`
variable when you need one stream.

### Panels

- **Overview** — active paths, publishing (ready) paths, total readers, active
  sessions across all protocols, inbound and outbound bitrate.
- **Throughput** — aggregate in/out bitrate over time; top 10 paths by inbound
  bitrate.
- **Sessions & Readers** — active sessions by protocol (RTSP, RTSPS, RTMP,
  WebRTC, HLS, SRT); readers by type.
- **Stream Quality** — inbound RTP packet loss (RTSP + WebRTC), inbound RTP
  jitter, frames discarded / in error.
- **Per-Path Detail** — sortable table: state, readers, in/out bitrate per path.

### Variables

- `datasource` — pick any Prometheus, so the dashboard is not pinned to one.
- `path` — multi-select, filters every panel to specific streams.

### Setup

Enable metrics in `mediamtx.yml`:

```yaml
metrics: yes
metricsAddress: :9998
```

Then scrape it, listing every node under one job to get the fleet view.

One gotcha worth knowing before you debug it: MediaMTX's default internal auth
only permits the `metrics` action from `127.0.0.1`/`::1`. A Prometheus on another
host scrapes from a non-localhost IP and gets back
`{"status":"error","error":"authentication error"}` instead of metrics. Grant the
`metrics` action to your Prometheus source IP — the repo has the exact config
block.

The repo also ships a one-command demo stack (MediaMTX + a synthetic ffmpeg
stream + Prometheus + Grafana, `docker compose up -d`) if you want to see the
panels populated before wiring it to production.

Source, demo stack, and metric notes: <REPO_URL>
MIT licensed — copy panels out of it freely.
```

### Registry fields

| Field | Value |
| --- | --- |
| Data source | Prometheus (required) |
| Minimum Grafana version | 10.0 — dashboard schema v39; tested on 10.x and 11.x |
| MediaMTX requirement | any version with the metrics endpoint (`metrics: yes`) |
| License | MIT |
| Repository | `<REPO_URL>` |
| Screenshot | `images/overview.png` — the registry listing wants at least one; the README screenshot block is still commented out, so this needs capturing from the demo stack first |

### Metrics the dashboard depends on

Listing these makes the compatibility question answerable without importing:

- Paths — `paths`, `paths_readers`, `paths_inbound_bytes`,
  `paths_outbound_bytes`, `paths_inbound_frames_in_error`
- Sessions / connections — `rtsp_sessions`, `rtsps_sessions`, `rtmp_conns`,
  `srt_conns`, `webrtc_sessions`, `hls_sessions`
- Quality — `rtsp_sessions_inbound_rtp_packets_lost`,
  `rtsp_sessions_inbound_rtp_packets_jitter`,
  `webrtc_sessions_inbound_rtp_packets_lost`,
  `webrtc_sessions_inbound_rtp_packets_jitter`,
  `webrtc_sessions_outbound_frames_discarded`,
  `hls_muxers_outbound_frames_discarded`

### Tags

```
mediamtx, streaming, rtsp, webrtc, srt, rtmp, hls, prometheus, video, media-server
```

First five match the tags baked into `dashboard.json`; the rest are there because
they are what people actually type into the registry search.

---

## 2. Reply for mediamtx issue #3623 — "Prometheus Grafana Dashboard"

Context: opened Aug 2024 by @SijanC147, asking whether a prebuilt dashboard exists
and saying they aren't comfortable building one. One comment, from @tomtom215 in
Sep 2025, confirming no community dashboard exists. Open and unlocked — this one
can be posted as-is. Lead with the aggregation problem, because that is the part
that makes a hand-built MediaMTX dashboard annoying and is the specific reason the
question keeps going unanswered.

```markdown
There still isn't one in the community registry, so I published the one I'd built
for my own servers: <REPO_URL>

The thing that makes this awkward to build by hand — and probably why nobody had
posted one — is that MediaMTX exports *per-entity* metrics rather than
pre-aggregated ones. Every path, session and connection is its own labeled series
with a constant value of `1`, so `paths` on a box with 200 streams gives you 200
series and a useless graph. The queries you actually want look like:

```promql
# live path count, not 200 flat lines
count(paths)

# how many are actually publishing
max by (name, state) (paths)

# bitrate — the byte metrics are cumulative counters
sum(rate(paths_inbound_bytes[$__rate_interval])) * 8

# sessions across all protocols
count(rtsp_sessions) + count(rtsps_sessions) + count(rtmp_conns)
  + count(webrtc_sessions) + count(hls_sessions) + count(srt_conns)
```

One detail that cost me time and isn't obvious from the error: `count()` on an
empty result returns nothing rather than `0`, so a stat panel goes to "No data"
the moment a path stops instead of showing zero. Every count in the dashboard is
`... or vector(0)` for that reason.

The dashboard is 14 panels in five sections — overview stats, aggregate and top-10 bitrate,
sessions by protocol, readers by type, RTP loss/jitter, and a per-path table —
with a multi-select `path` variable and a datasource variable, so it works for one
server or for a fleet scraped by one Prometheus.

There's a `docker compose up -d` demo stack in the repo (MediaMTX + a synthetic
ffmpeg stream + Prometheus + Grafana) if you want to see it with data before
pointing it at anything real. Import is the normal Dashboards → New → Import with
`dashboard.json`.

One thing to check if the panels come up empty against your own server: MediaMTX's
default internal auth only allows the `metrics` action from `127.0.0.1`/`::1`, so a
Prometheus running anywhere else gets
`{"status":"error","error":"authentication error"}` from `/metrics` rather than a
scrape failure you'd notice. The repo's `mediamtx.yml` has the block that grants
it to a specific source IP.

MIT, so take panels out of it if you only want part of it. Happy to add SRT link
stats or alert rules if anyone has a real deployment to check them against —
that's the part I can't test alone.
```

---

## 3. Reply for mediamtx issue #163 — "Prometheus Metrics into Grafana dashboard?"

**#163 is closed and locked** (auto-locked 2023-01-01). Comments are disabled, so
this draft cannot be posted to that thread as-is. Three ways to get it in front of
the same audience, best first:

1. **Ask a maintainer to unlock #163**, then post the draft below verbatim. It is
   written for that thread — @benbullnz asked for a template in 2020 and got a
   Prometheus scrape config instead, which is the gap this fills.
2. **Post it as a GitHub Discussion** (Discussions are enabled on the repo) titled
   something like "Grafana dashboard for the Prometheus metrics", opening with
   "#163 asked for this in 2020 and #3623 again in 2024 — here it is." A
   cross-reference makes it show up as a linked mention on both threads without
   needing anything unlocked. **Recommended** — no permission needed, and it's the
   thread that keeps working as a landing spot.
3. Do nothing on #163 and rely on the #3623 reply, which is the live thread. The
   locked-issue traffic is search traffic, and the mention backlink from option 2
   captures that anyway.

Whichever route, this text should differ from the #3623 reply — same repo, but
these are different questions. #163's audience got as far as "Prometheus is
scraping, now what", so this one leads with what to do with the data rather than
with the aggregation problem.

```markdown
Coming back to this one late — this got answered with a scrape config, but the
actual question was for a dashboard template and that part never got a reply.
Here's one: <REPO_URL>

It imports straight into Grafana against the Prometheus you set up in this thread
(Dashboards → New → Import, upload `dashboard.json`, pick your datasource). 14
panels: path and session counts, in/out bitrate, sessions broken down by protocol
— RTSP, RTSPS, RTMP, WebRTC, HLS, SRT — readers by type, inbound RTP packet loss
and jitter, and a per-path table with state, reader count and bitrate. A
multi-select `path` variable scopes all of it to one stream when you're chasing a
specific problem.

The metric surface has grown a lot since 2020, so the two things worth knowing if
you're building your own instead:

- The byte metrics (`paths_inbound_bytes`, `paths_outbound_bytes`) are cumulative
  counters, so bitrate is `rate(...) * 8`, not the raw value.
- The entity metrics (`paths`, `rtsp_sessions`, `webrtc_sessions`, …) emit one
  series per live entity with value `1`. `count()` gives you the live total;
  graphing them directly gives you one flat line per stream.

If you want to see it working before touching your server, the repo has a
`docker compose up -d` stack — MediaMTX, a synthetic ffmpeg stream, Prometheus and
Grafana, dashboard provisioned — that populates in about a minute.

MIT. Panels are worth stealing individually if the whole thing isn't what you
want.
```

---

## Pre-post checklist

- [ ] Repo pushed to a public remote; `<REPO_URL>` is real and resolves
- [ ] `github.com/<you>/` placeholder replaced in README.md
- [ ] `images/overview.png` captured from the demo stack; README screenshot block
      uncommented (the registry listing wants a screenshot)
- [ ] `docker compose up -d` verified from a clean clone at that URL — both replies
      promise it works
- [ ] Registry listing published; note the assigned `<GRAFANA_ID>`
- [ ] #3623 reply posted
- [ ] #163 route chosen (unlock / Discussion / skip) and executed
