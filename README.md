# MediaMTX Grafana Dashboard

A clean, **fleet-ready** Grafana dashboard for [MediaMTX](https://github.com/bluenviron/mediamtx), built on its native Prometheus metrics endpoint. Ships with a one-command demo stack (MediaMTX + a synthetic stream + Prometheus + Grafana) so you can see it working in under a minute.

> MediaMTX exposes **per-entity** metrics — every path, session, and connection is its own labeled series. This dashboard does the aggregation for you (`count()` the entities, `rate()` the byte counters into bitrates, `sum by (state / readerType)`), so it works for a single node **or** a whole fleet of nodes scraped by one Prometheus.

<!-- Screenshot: run the demo stack below, then capture the dashboard into images/overview.png and uncomment:
![MediaMTX Fleet Overview dashboard](images/overview.png) -->

## Quick start (demo)

```bash
git clone https://github.com/<you>/mediamtx-grafana-dashboard
cd mediamtx-grafana-dashboard
docker compose up -d
```

Then open **http://localhost:3300** (login `admin` / `admin`) → **Dashboards → MediaMTX → MediaMTX — Fleet Overview**. A synthetic 720p test stream is published automatically, so the panels populate immediately.

> Grafana is exposed on **3300** (not its default 3000) to avoid colliding with common dev servers. Prometheus is on `:9090`, MediaMTX metrics on `:9998`.

Tear down with `docker compose down`.

## Use with your own MediaMTX

1. **Enable metrics** in your `mediamtx.yml`:

   ```yaml
   metrics: yes
   metricsAddress: :9998
   ```

   > **Gotcha:** MediaMTX's default internal auth only permits the `metrics`
   > action from `127.0.0.1` / `::1`. A Prometheus on another host/container
   > scrapes from a non-localhost IP and gets `{"status":"error","error":"authentication error"}`.
   > Grant the `metrics` action to your Prometheus source IP (see the
   > [`mediamtx.yml`](mediamtx.yml) in this repo for the exact block; tighten
   > `ips:` to your Prometheus host in production).

2. **Scrape it** from your Prometheus:

   ```yaml
   scrape_configs:
     - job_name: mediamtx
       metrics_path: /metrics
       static_configs:
         - targets:
             - node-1.internal:9998
             - node-2.internal:9998   # list every node for a fleet view
   ```

3. **Import the dashboard**: in Grafana, **Dashboards → New → Import**, upload [`dashboard.json`](dashboard.json) (or paste its contents), and pick your Prometheus data source when prompted.

## What's on it

| Section | Panels |
| --- | --- |
| **Overview** | Active paths, publishing (ready) paths, total readers, active sessions (all protocols), inbound & outbound bitrate |
| **Throughput** | Aggregate in/out bitrate over time; top 10 paths by inbound bitrate |
| **Sessions & Readers** | Active sessions by protocol (RTSP, RTSPS, RTMP, WebRTC, HLS, SRT); readers by type |
| **Stream Quality** | Inbound RTP packet loss (RTSP + WebRTC); inbound RTP jitter; frames discarded / in error |
| **Per-Path Detail** | Sortable, filterable table: state, readers, in/out bitrate per path |

A **Path** template variable (multi-select) filters the whole dashboard to specific streams; a **Data source** variable lets you point it at any Prometheus.

## Notes on the metrics

- Byte counters (`paths_inbound_bytes`, etc.) are cumulative — the dashboard applies `rate(...) * 8` to show **bitrate in bits/sec**.
- Entity gauges (`paths`, `rtsp_sessions`, `webrtc_sessions`, …) are `1` per live entity, so `count()` gives the live total. Empty results fall back to `0` via `or vector(0)`.
- Jitter is shown in the metric's raw units as reported by MediaMTX (unscaled).
- Full metric reference: [MediaMTX metrics docs](https://github.com/bluenviron/mediamtx/blob/main/docs/2-features/23-metrics.md).

## Compatibility

- **Grafana** 10.x / 11.x (dashboard schema v39).
- **MediaMTX** with the metrics endpoint enabled (metric names verified against current docs, July 2026).
- **Prometheus** any recent version.

## Contributing

Issues and PRs welcome — extra panels (SRT link stats, MoQ sessions), alert rules, and recording-panel additions are all fair game.

## License

[MIT](LICENSE).
