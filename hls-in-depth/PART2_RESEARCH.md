# References:

- https://bozhang-26963.medium.com/a-quick-latency-comparison-of-apple-ll-hls-and-the-community-driven-lhls-e3eb3e7447ee

## Mux blog: https://www.mux.com/blog/the-community-gave-us-low-latency-live-streaming-then-apple-took-it-away

- > generally the lower limit of segment duration has been found to be around 2 seconds, which will deliver a passable streaming experience
- Important features:
    - Partial segments
    - HTTP/2 Pushed Segments
        - In traditional HLS we make two round trips, one for fetching playlist, next is fetching the segments. Apple tried to reduce these HTTP calls by allowing server to push content to client, but as we know this did get scraped later.
    - Blocking playlist requests
        - Do not send me playlist until I (client) receives a particular, future to be media segment
    - Playlist delta updates
        - An interesting point, better to read directly in article
        - Allow quicker switching between resolutions/bitrates(renditions in short) by sending links to some HEAD chunks of other renditions. This in reality does not allow one to directly jump to another rendition, instead it is an optimization to start fetching playlist tad bit early and leverage HTTP/2 to get chunks.
    - Faster bitrate switching
        - __Re-read this part__
        - Questions:
            - How does a client know what media segment to block on when sending a blocking playlist update and getting the new segments at the same time with HTTP/2 push?
- A lot of state needs to be handled now. Earlier playlist didn't need to know care about client a lot, but now it needs to know a bunch of stuff to even work.
- It relies a lot on query params, and usually a part or all of query params are encrypted to not allow malicious users to access it. With query params getting included in the core of the protocol and encryption being present, this makes the situtaion a lot more tricky for caching implementations.
- With blocking playlist feature combined with last point, makes implementing CDN super hard. One definitely can't use current HTTP CDNs, and would have to build custom ones or add extension functionality to them to get them working somehow, both seem to be a lot of effort.
- CL-HLS is deprecated: https://github.com/video-dev/hlsjs-rfcs/pull/1#issuecomment-609840654 . PR to revert it: https://github.com/video-dev/hls.js/pull/2864
- > LHLS uses the same strategy used when delivering low latency MPEG DASH - HTTP 1.1 chunked transfer encoding.
- BWE is also a big problem in LL-HLS, because of blocking playlist and chunked transfer being used.

## A quick latency evaluation of Apple LL-HLS and the community-driven LHLS: https://bozhang-26963.medium.com/a-quick-latency-comparison-of-apple-ll-hls-and-the-community-driven-lhls-e3eb3e7447ee

- > CL-HLS uses chunked encoding (video segments are divided into CMAF chunks or partial TS segments) and HTTP Chunked Transfer Encoding (CTE) to enable early segments delivery before whole segments are generated.
- > Note that CTE is only used by LL-CMAF and the CL-HLS. Apple LL-HLS uses the new EXT-X-PRELOAD-HINT tag (a standard HLS tag) to enable early segment delivery.
- > The CL-HLS uses a non-standard tag, EXT-X-PREFETCH to publish in-progress video segments for clients to prefetch.
- In CL-HLS partial segments are known as prefetch segments. N prefetch segments make up a complete segment. This tage also makes whole CL-HLS spec backwards compatible to HLS
- Article writes: "For example, since a player has to explicitly request each chunk in a segment, it can accurately measure the download bandwidth and adapt bitrate accordingly.", that seems wrong cause server can always HTTP2 Push right? Should re-check this with spec

## What’s the delay? Pushing the limits of ultra-low latency HLS streaming: https://www.nativewaves.com/news/press-releases-and-articles/whats-the-delay-pushing-the-limits-of-ultra-low-latency-hls-streaming

- > Simply put, Community LHLS advertises the most recent (edge) media segment to the receiver while it is still being produced, whereas Apple LLHLS advertises only those portions of it which are already available.
- > Most prominently, Community LHLS breaks Adaptive Bit Rate (ABR) streaming due to chunked transfer and is already phased out in most implementations in favor of Apple LLHLS.
    - Most probably due to same BWE problems we discussed in mux article
- According to article, LL-HLS won over L-HLS because of it not providing ABR

## Low Latency Streaming With HLS: All You Need to Know: https://www.muvi.com/blogs/low-latency-streaming

- > Apple revealed an update to the LL-HLS draft standard at the start of 2020. The upgrade eliminated the necessity for HTTP/2 Push and instead introduced a new tag to indicate future segments in response to industry concerns and conflict.
    - HTTP/2 Push was replaced by preload hints
- Reducing Latency in HLS:
    - Reducing the Segment Time
        - Smaller segment, smaller things to download, faster cycles, kinda bandwidth intensive
    - Lowering the Keyframe Interval
        - Lower keyframe enables lower segment, as each segment is a GOP. A single segment can have mulitple keyframes. But lowering keyframe down to 1 in a single segment and then making GOP smaller, allows us to make segments smaller
    - Increasing the Number of Video chunks
        - In newer protocols media is chunked before sending, if we increase the number of chunks, we would have seemingly faster cycle from segmenter to distributor. Similar reasoning as smaller segment size. And hence again playlist may need to be fetched very frequently so that becomes a problem, but that is also addressed by prefetch tags now.
    - Announcing Segments Beforehand
        - Reasoning here is, video players(clients) need to first read playlist, understand what more segments will be needed in upcoming segment fetch, allocate/lay out memory for them. This needs to be done instantly on spot in vanilla HLS. But in low-latency versions you can tell client that these will be the segments which you can pre-fetch. They can allocate memory faster and then start fetching segment one after the other. They will be available due to chunked transfer encoding. A HTTP chunks end is marked by sending an empty chunk.

## The Evolution of Low-Latency Video Streaming: https://www.brightcove.com/en/resources/blog/evolution-of-low-latency-streaming/

- > While LL-HLS and LL-DASH worked well in unconstrained network environments, they struggled in low bandwidth or highly variable networks (which are typical in mobile deployments). Observed effects included highly variable delays, inability to prevent buffering, and frequent bandwidth switches or inability to use the available network bandwidth. Some players simply switched into non-low-latency streaming under such challenging network conditions.

## LL-HLS, LHLS, DASH-LL: Challenges and Differences by Will Law: https://www.youtube.com/watch?v=DVrPv-8PUm4

- Various protocols:
    - LL-HLS = AL-HLS = Apple HLS
    - CL-HLS = L-HLS = Community's/JWT HLS
    - LL-DASH

- Similarities:
    - All of these are chunked protocols, each chunk is a moov and mdat pair.
    - Support E2E latencies between 2-10 seconds range
    - Backwards compatible with older players
    - Cacheable by CDNs
    - Support DRM
    - Support Ad Insertion
    - Support multiple codecs type
    - Allow ABR Playback
    - HTTP Delivery

- Differences:
    - HTTP CTE: LL-HLS does not support this
    - Describe internal segment structure: Only LL-HLS supports this
    - Playlist refresh with each chunk: Only LL-HLS supports this
    - Objects delivered at line speed: Only LL-HLS
    - Need a single static server URL for playlist/segments: Only LL-HLS needs this, due to HTTP/2 and playlist fetch requirements
    - HTTP/2: Only LL-HLS
    - Smart Origin to modify playlists: Only LL-HLS
    - Deterministic start-up: Only LL-HLS

- CTE allows chunks to start getting delivered as soon as they are ready on encoder level. For getting a segment you only make one request and you start receiving chunks. A chunk is roughly of 0.33s size (= 1 frame), hence a 6 second segment would have 180 chunks. So now, bottleneck becomes encoder itself.
 - > Data speed is encoder limited, versus line-speed limited. This makes it difficult for estimate the throughput by timing the receipt of the object.

- Challenges with LL-HLS:
    - Origin intelligence
    - increase in request rate from each client
    - CDNs with PUSH support
    - Ad-insertion support or doing things like regionalization, a chunk window is small(320ms) and doing regionalization in that is a challenge in itself

- In CL-HLS and LL-DASH BWE is a problem because let's say we make a request for a segment, we start receiving it, we don't know yet how big it is, how may chunks we will receive int it, so we can't estimate bandwidth properly.
- Only LL-HLS provides IDR (internal segment descriptions). Independent segments/chunks are helpful for clients to know which point can they start decoding to start the stream. Now buffer size is client dependent, some may say we want 2 seconds of data loaded before we wanna start, some may say 1. So this description helps clients in understanding which chunk can they start decoding from and start filling their buffer.

- Bunch of chunks = fragement
- Bunch of fragments = segment

## https://www.theoplayer.com/blog/evolution-of-ll-hls

- > It was Twitter’s Periscope platform that first implemented a number of these improvements and bundled them into LHLS in 2016.
- > As I-frames are significantly larger than predicted frames (P-frames), reducing the segment size (and adding in more I-frames) would increase the overall bandwidth used.
- > For SSAI, the specification required an intimate collaboration between the playlist manipulator and the CDN.
    - SSAI: Server Side Ad Injection
- > In early 2020, Apple announced an update to the draft LL-HLS specification. After the concern and friction within the industry, the update removed the HTTP/2 Push requirement and instead introduced a new tag to announce upcoming segments.
- New Tag: #EXT-X-PRELOAD-HINT
- With new RFCs out, one can also use delta playlists and blocking playlist reloads in traditional HLS.
- > Blocking playlist reload is no longer mandatory, but is recommended.
- > Multiple preload hints can be listed for the same type.
- > It is no longer defined how long parts must remain in the playlist (parts must be removed after 3 target durations, and should appear at "live edge", but "live edge" is not defined).

## https://www.theoplayer.com/blog/low-latency-chunked-cmaf

## https://www.theoplayer.com/blog/low-latency-hls-lhls

- Two imp. things it does to improve latency:
    - Leveraging HTTP/1.1 chunked transport for segments
    - Announcing  segments before they are available

- > players which are not LHLS aware can still play the stream as if it was a normal HLS stream and still get an improvement in latency.

## https://aws.amazon.com/blogs/media/alhls-apple-low-latency-http-live-streaming-explained/

- > Underpinning both solution is the ability for the client side player to request a segment that is still being encoded. In other words, the player is able to decode the top of the segment while the tail of the segment is still being generated.
    - Here both solution refers to CTE and Chunked CMAF
- > #EXT-X-PARTs with byte-range offset as the locations rather than discrete files as in the Apple example above.
- Let's say CDN cache is set at 2 seconds, what will happen is CDN will not serve new segments produced during this 2 seconds (each chunk is generated at 333 ms roughly). To let clients get latest info, they can use _HLS_msn and _HLS_part tags. These allow for cache busting and blocking playlist reloads:
    - > _HLS_msn=<N>which instructs the origin that the client is only interested in a playlist that contains the media sequence number N and;
    - > _HLS_part=<M>which instructs the origin that the client is only interested in a playlist that contains partM of media sequence N.
- > Apple have now also introduced HTTP2 Server Push into the HLS specification. Essentially, the client should pass again via query string, this time a boolean of _HLS_push=1/0, whether or not the most recent partial segment at the bottom of the m3u8 list should be pushed in parallel with the m3u8 response.
- > That is clearly why you can now signal via the query string _HLS_skip=YES which instructs the server to only send the delta from the last playlist to now. The resulting manifest will insert #EXT-X-SKIP:SKIPPED-SEGMENTS=3 in lieu of the actual segments; in this case 3. The #SERVER-CONTROL attribute ofCAN-SKIP-UNTIL= should also be set to a horizon of no less than 6 segments from the live edge.
- > To this end, you can now include ?_HLS_report=/other/manifest.m3u8in the request. This can be used to include the segment availability hints of adjacent representations to the one currently requested. Switching renditions are defined via the INDEPENDENT=YES attribute.

## https://cloudinary.com/guides/live-streaming-video/low-latency-hls-ll-hls-cmaf-and-webrtc-which-is-best

- > CMAF defines the following logical media objects:
    - > CMAF track, which contains encoded samples of media, such as video, audio, and subtitles, with a CMAF header and fragments. The samples are stored in a CMAF-specified container based on the ISO Base Media File Format (ISO BMFF). You can also protect media samples by means of MPEG Common Encryption (COMMON ENC).
    - > CMAF switching set, which contains alternative tracks with different resolutions and bitrates for adaptive streaming, which you can splice in at the boundaries of CMAF fragments.
    - > Aligned CMAF switching set, which contains switching sets from the same source through alternative encodings (e.g., with different codecs), which are time-aligned to one another.
    - > CMAF selection set, which contains switching sets in the same media format. That format might contain different content, such as alternative camera angles or languages; or different encodings, such as alternative codecs.
    - > CMAF presentation, which contains one or more presentation time-synchronized selection sets.

## https://bitmovin.com/live-low-latency-hls/

- > Partial segments can also reference the same file but at different byte ranges. Clients can thereby load multiple partial segments with a single request and save round-trips compared to making separate requests for each part.
- > Soon to be available partial segments are advertised prior to their actual availability in the playlist by a new EXT-X-PRELOAD-HINT tag.
- > A new EXT-X-SKIP tag replaces the content of the playlist that the client already received with a previous request.
- > When playing at low latencies, fast bitrate adaptation is crucial to avoid playback interruptions due to buffer underruns. To save round-trips during playlist switching, playlists must contain rendition reports via a new EXT-X-RENDITION-REPORT tag that informs about the most recent segment and part in the respective rendition.

## Optimizing LL-HLS: 4 Recommendations For The Best Low-Latency streaming: https://www.theoplayer.com/blog/optimizing-ll-hls-4-recommendations

- GOP: Set your keyframe interval to 2-3 seconds
- Part Size: Use 400msec Part Size in for the lowest end-to-end latency
- Segment Size: Set it equal to or larger than your GOP size
- Buffer Size, Network Tolerate & ABR: Find the best middle ground

## General:

- GOP is the distance between two keyframes, measured in terms of number of frames, or the amount of time between keyframes.

## My notes:

- LL-HLS Highlights:
    - Partial Segments Deliver: Partial TS + CMAF
    - HTTP/2 Push: Now not necessary, can use #EXT-X-PRELOAD-HINT tag instead
    - Blocking Playlist Reloads: Not necessary
    - Playlist Delta Updates: Now ported back to HLS too
    - Fast Bitrate Switching
    - Easier ABR
    - Nothing using CTE: Could have been super helpful, but ABR would have been slightly hard
    - Rendition Reports
- LL-HLS is all about optimization around getting segments from encoder to client as soon as possible or in more efficient manner.
- Buffer of three segments is still present. Just that filling those buffer gaps is much more easier and continuous process, rather than waiting periodic poll of manifest file after a complete segment is received.

- In CL-HLS prefetch tag only pre-advertised(yet to be formed) full segments, these were then started to being fetched using CTE.
- In LL-HLS chunks are explicitly written in the manifest file and then fetching happens. This is also the reason they needed a quick way to send these chunks, and hence HTTP/2 Push. If we don't use HTTP/2 push, we have to make a bunch of requests for manifest file updates.

## Rough Section:

========================================================

Next things to write:
- Playlist deltas, how that helps in bandwidth reduction
- Blocking playlist updates
    - talk about cache busting too
- Rendition Reports
- Faster renditions faster
- Mention exact tags which make all of the above talked things possible
- Discuss a LL-HLS Playlist
    - made either from apple tools or downloaded from online samples

If we want to obtain low-latency, there are few areas we can focus upon, waiting for a 6 second segment upfront seems like a heavy cost.

When trying to optimize for glass-to-glass latency, we have three main areas to focus upon,
