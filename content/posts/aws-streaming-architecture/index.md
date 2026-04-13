+++
date = '2026-04-13T07:51:27+03:00'
draft = false
title = 'Broadcasting Video on AWS: Architecture, Options, and Cost Math'
+++

*A technical deep-dive into how bits move from encoder to viewer — and what it costs.*

---

## Introduction

Streaming video at scale is, at its core, a bandwidth and caching problem. Every architectural decision — which AWS services to use, how long your segments are, how much delay you're willing to tolerate — has a direct, calculable effect on your monthly bill. For a hands-on way to estimate these numbers, check out our interactive <a id="dashboard-link" href="/posts/aws-streaming-architecture/streaming-dashboard.html">AWS Streaming Cost Dashboard</a>. This post walks through the full picture: how the pipeline works, the three main deployment patterns, and the cost factors in each one.

<script>
  document.addEventListener('DOMContentLoaded', function() {
    const link = document.getElementById('dashboard-link');
    if (link) {
      link.addEventListener('click', function(e) {
        e.preventDefault();
        const isDark = document.documentElement.classList.contains('dark') || 
                       document.documentElement.getAttribute('data-theme') === 'dark' || 
                       document.body.classList.contains('dark') ||
                       localStorage.getItem('theme') === 'dark' ||
                       localStorage.getItem('pref-theme') === 'dark';
        window.location.href = this.href + '?theme=' + (isDark ? 'dark' : 'light');
      });
    }
  });
</script>

---

## How the Pipeline Works

Regardless of mode, all video delivery on AWS follows the same fundamental stages.

**Ingest** is where raw video enters the system. A camera or encoder sends an RTMP or RTP stream to AWS MediaLive, which transcodes it into multiple quality renditions (an "ABR ladder") in parallel — typically 360p, 480p, 720p, and 1080p at bitrates ranging from 0.8 Mbps to 6 Mbps.

**Packaging** is handled by AWS MediaPackage, which takes the transcoded streams and wraps them into delivery formats — primarily HLS (HTTP Live Streaming) or DASH. MediaPackage also handles the DVR buffer, allowing viewers to rewind within a rolling time window. It acts as the "origin" that CloudFront pulls from.

**Distribution** is CloudFront's job. With over 400 Points of Presence (PoPs) worldwide, CloudFront caches video segments close to viewers, serving the vast majority of requests without touching the origin at all. The efficiency of this caching — the cache hit rate — is the single most important variable in your cost equation.

**Playback** happens on the viewer's device. An ABR player (like HLS.js or Video.js) continuously monitors download speed and buffer health, switching between quality levels to maintain smooth playback. Players request new segments on a fixed cadence — every 2 seconds for low-latency streams, every 6–10 seconds for standard streams.

---

## The Three Deployment Patterns

### Pattern 1: True Live (Low-Latency HLS)

Low-Latency HLS (LL-HLS) pushes end-to-end latency down to 1–3 seconds by using very short segments — 200ms to 2 seconds — combined with HTTP/2 server push. This is the approach used for live sports betting, synchronized social viewing, and real-time auctions.

The cost penalty is significant. Short segments expire from CloudFront's cache before many viewers can be served from it. If a 2-second segment is generated, uploaded to MediaPackage, and pulled by CloudFront, it may only serve viewers for 2–4 seconds before being superseded by the next segment. Cache hit rates are typically **40–70%**, meaning 30–60% of viewer requests become origin fetches — each one costing MediaPackage egress fees on top of CloudFront egress fees.

**When to use it:** Live sports, live auctions, gaming streams, any use case where viewers need to act on what they see within seconds.

### Pattern 2: Near-Realtime Live (Recommended Default)

A 2–5 minute delay unlocks dramatically better economics with almost no user-experience cost for most content types. The mechanism is simple: with 6–10 second HLS segments and a DVR buffer of a few minutes, segments have been "warm" in CloudFront's cache for several minutes by the time most viewers request them.

Consider the math: if your stream has 10,000 concurrent viewers and a segment is 8 seconds long, each segment gets requested approximately 10,000 times over its 8-second window. With a true live setup, the first 30–60% of those requests are cache misses. With a 5-minute delay, the segment has already been pulled and cached by the first viewers, so the cache hit rate jumps to **85–92%** — approaching VOD-level efficiency.

MediaLive still runs continuously (it's a live encoder), so that cost is identical to true live. But MediaPackage origin fees drop by 3–5×, and overall infrastructure cost falls by roughly 30–50% compared to LL-HLS.

**When to use it:** Live news, sports (non-betting), concerts, corporate all-hands, webinars — any content where viewers are watching "as it happens" but don't need sub-5-second synchronization.

### Pattern 3: Pure VOD

Pre-recorded content stored in S3 and delivered through CloudFront is the most cost-efficient pattern. There is no MediaLive or MediaPackage in the pipeline — the encoder runs once at production time, outputs MP4 or HLS files, and they go into S3. From there, CloudFront serves them indefinitely.

Cache hit rates of **92–99%** are typical for popular content, because viewers are spread across time. Viewer #1,000 watching a popular episode benefits from the cache warming done by viewers #1 through #999 across all previous sessions. S3 is only hit on the very first request from each CloudFront edge node.

Optional **Origin Shield** adds a regional aggregation layer between CloudFront's 400+ PoPs and S3. Instead of each PoP independently fetching from S3 on a cache miss, they all funnel through a single Origin Shield node per region. This reduces S3 GET requests by 10–50× and costs approximately $0.012/GB of Origin Shield traffic — usually a net saving.

**When to use it:** Any pre-recorded content — training videos, product demos, movies, TV shows, recorded webinars.

---

## Cost Components Explained

### CloudFront Egress

This is the dominant cost in every pattern. CloudFront charges per GB of data delivered to viewers, with rates varying significantly by region (for example, US and Europe are typically cheapest, while South America and parts of Asia are higher).

Committed use discounts (via custom pricing agreements) can reduce these rates above certain volume thresholds. At enterprise scale, per-GB rates can drop substantially.

### AWS MediaLive

MediaLive is a managed live video transcoding service. It runs 24/7 during a live event — or all month if you're running a continuous channel — regardless of how many viewers are watching. Pricing is calculated as an hourly rate that depends on the output resolution (from SD to 4K) and codec (e.g., H.264 vs H.265). Higher resolutions and more advanced codecs command a higher hourly premium.

A typical multi-bitrate channel with multiple renditions is priced based on the highest output resolution rather than being multiplied per rendition.

This fixed cost means MediaLive dominates your bill at low viewer counts, but becomes a negligible percentage of the total cost at high viewer counts.

### AWS MediaPackage

MediaPackage charges for data that originates from it — specifically, the volume of video that CloudFront pulls from MediaPackage when it doesn't have a segment cached. This is a per-GB origin output cost.

This cost is directly tied to your cache miss rate. With true live patterns driving higher cache misses, MediaPackage costs will represent a significant line item. With near-realtime, this footprint shrinks dramatically, and with VOD (which doesn't use MediaPackage), it naturally falls to zero.

### S3 Storage and GETs (VOD)

For VOD, your content lives in S3:

- **Storage:** You pay a standard per-GB monthly fee to store your video library.
- **GET requests:** S3 charges for requests, but with high cache hit rates at the CDN layer, S3 GET costs are generally negligible even at scale.
- **Data transfer out (to CloudFront):** Data transfer from S3 to CloudFront is free. You only pay for the final egress from CloudFront to the viewer.

### Origin Shield (VOD, optional)

Origin Shield sits between CloudFront's edge nodes and your S3 origin. When enabled, cache misses from all 400+ CloudFront PoPs are aggregated through a single regional Origin Shield node, which either serves from its own cache or makes a single request to S3.

- **Cost:** You pay a small fee per GB of data passing through the Origin Shield.
- **Saving:** It significantly reduces S3 GET requests, marginally improves the cache hit rate, and reduces latency for the first viewers.
- **Net effect:** At higher traffic volumes with large libraries, the lower S3 request costs typically offset the Origin Shield fee, making it a net saving.

---

## The Cache Hit Rate: Why It Dominates Everything

Cache hit rate is the variable with the highest leverage in your cost model. Every percentage point of cache miss translates directly into origin fetch costs on top of the CDN egress costs you're already paying.

Here's the full cost multiplier by pattern, relative to pure CloudFront egress:

| Pattern | Cache hit rate | Origin fetch overhead | Effective cost multiplier |
|---------|---------------|----------------------|--------------------------|
| True live (LL-HLS, 2s segments) | 40–65% | High | 1.4–1.8× |
| Near-realtime (5 min delay, 8s segments) | 85–92% | Low | 1.05–1.15× |
| Pure VOD (S3 origin) | 92–99% | Negligible | 1.01–1.04× |

The "cost multiplier" compounds with scale. At 1 PB/month of delivery, the difference between 50% and 90% cache hit rate is hundreds of thousands of dollars in MediaPackage and origin infrastructure costs.

---

## Key Optimization Levers

**Codec efficiency** is the highest-impact lever and it applies upstream, before CloudFront. Switching from H.264 to H.265 reduces required bitrate by ~40–50% at equivalent quality. AV1 reduces it a further 20–30%. At scale, halving your average bitrate brings massive permanent savings in CloudFront egress alone.

**Segment duration** is the primary control over cache hit rate in live scenarios. Moving from 2-second (LL-HLS) to 8-second segments, combined with a 5-minute delay, is the single cheapest improvement available to a live broadcaster: it requires a one-line configuration change in MediaPackage and drastically cuts origin costs.

**Committed use pricing** kicks in via custom CloudFront pricing agreements. AWS typically offers tiered discounts starting at certain volume thresholds. If your delivery volume is predictable, negotiate a contract — on-demand pricing is the most expensive option at scale.

**Multi-CDN** routing (e.g., combining CloudFront with Fastly or Akamai) introduces price competition and reduces single-vendor dependency. Most large streaming platforms use 2–3 CDNs simultaneously, dynamically routing traffic based on cost and performance.

**Regional distribution** matters if your audience is concentrated. A US-only audience typically enjoys the lowest baseline rates. A global audience with heavy traffic in certain regions (like parts of Asia Pacific and South America) faces significantly higher per-GB costs. Serving a local CDN or regional caching tier in high-cost regions can significantly reduce blended delivery rates.

---

## Summary Comparison

| | True Live | Near-Realtime (5 min) | Pure VOD |
|--|-----------|----------------------|----------|
| **Latency** | 1–3 seconds | 2–5 minutes | On-demand |
| **Segment size** | 200ms–2s | 6–10s | N/A |
| **Cache hit rate** | 40–70% | 85–92% | 92–99% |
| **MediaLive** | Yes (fixed) | Yes (fixed) | No |
| **MediaPackage** | Yes (high cost) | Yes (low cost) | No |
| **S3** | Minimal | Minimal | Primary storage |
| **Relative cost** | Highest | Medium | Lowest |
| **Cost multiplier vs VOD** | 1.8–2.5× | 1.2–1.5× | 1.0× |
| **Best for** | Sports betting, live auctions | News, sports, events | Training, entertainment |

---

## Conclusion

The architecture decision tree is simpler than it might appear. Start with the minimum acceptable latency for your use case. If you can tolerate 2–5 minutes of delay, choose the near-realtime pattern — you get 85–92% of VOD's cost efficiency with a live product experience. If you need sub-5-second latency, budget for LL-HLS and optimize aggressively on codec and committed pricing. If your content is pre-produced, VOD on S3 + CloudFront is the default answer.

The cost math is multiplicative: total egress volume × cache miss rate × origin fetch rate × codec overhead. Attack each multiplier independently, and the compounded savings are substantial.

---

*Regional rates and committed use discounts vary. Always validate against the [AWS Pricing Calculator](https://calculator.aws/pricing/2/home) for production estimates.*
