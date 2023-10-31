# Optimizing HLS for Latency: Understanding LL-HLS and CL-HLS

## Introduction

In [HLS in depth](https://dyte.io/blog/hls-in-depth/) blog, we explored core HLS protocol, how it works, its plus points and its limitations. When discussing limitations, one notable point was higher latency. Reducing it is a tricky problem, and hence efforts in this direction began, today we will explore how they led to the final evolution of LL-HLS and CL-HLS.

## Terminology

Before we start, let's get our terminology straight. In this article CL-HLS refers to Community HLS, which is also known as L-HLS. And LL-HLS refers to Apple's low latency HLS or AL-HLS. I have seen a lot of confusion around these (personally had much more). In this article we will use CL-HLS and LL-HLS as the terms to refer to both independent low latency solutions.

## HLS

With traditional HLS, we wait for a segment to be produced. Then we copy it to some web-server/edge location which then gets polled by client to fetch them. For a 6 second segment, this would mean 6 second input wait + encoder costs + CDN fetch costs + client buffer, which is 3 segements, so 4 segements to start playing i.e. 24 seconds. One case see how latency start to add up in traditional HLS.

Before even starting to optimize, let's get a mental model around this:

<-- Diagram of whole flow -->
(
    Source -> Encoder -> Web server -> CDN -> Client
)

## What we don't focus to optimize

One would say we can start optimizing from source to encoder delivery, the catch here is HLS Spec does not talk about this part of delivery, so it can be done in any way open to implementor. Most famous way is to use RTMP, but we have been seeing WebRTC emerging as a nice alternative. With OBS getting first class support of it using WHIP, it is very interesting to see how ingest side of things evolve.

But for optimizations in HLS, we don't focus on ingest from spec perspective. So next part is encoder, again one can get the best encoder and it's a complete field of it's own to optimize them, also the reason spec doesn't talk about them explicitly. So that is out of the picture.

But now everything after segments leaving encoder is HLS territory(including client) and open for optimizations. So our problem statement is getting output from encoder to glass(viewer's screen) as fast as possible.

## Segment Size Optimization

If we revisit the segment flow mentioned above, one could say we can just segment size smaller, less than a second maybe and then keep sending them. But there's a catch in that, we always want each segment to be independently decodable. What do I mean by independently decodable? I am going to give a small, just enough primer on how encoder works, everything above that is homework for you :P

When talking about still pictures, we know one way they can be represented in memory and that is RGB. So we store RGB value for each pixel and a nested array of them forms a picture! That's a lot of data points! Well yes, now let's say if we are dealing with a stream of pictures, and we store this representation for each one of them, that would need a lot of memory. So what we do is we choose a base representation, we can call it I-Frame(keyframes), they are not directly RGB representation as we mentioned, but with some smart compression. Now whatever changes in next frame, we take a smart difference of it from I-Frame and store it to as P-Frame. P-Frames hence are dependent frames and I-Frames hence are independent frames, they don't depend on anyone else. In [wikipedia's](https://en.wikipedia.org/wiki/Video_compression_picture_types) words:

```
 I‑frames are the least compressible but don't require other video frames to decode. P‑frames can use data from previous frames to decompress and are more compressible than I‑frames.
```

<-- Diagram for I-Frame and P-Frame -->

Back to our topic, each segment needs atleast one I-Frame, a simple reason being, a viewer can join stream any point in time and peer starts getting segment from that point of time, hence that segment should be independent to start the stream. Now we discussed I-Frames are heavy, and hence us reducing segment size would cause keyframes generation to increase and size of each segment to increase. Segment size directly corresponds to bandwidth usage, which would start choking on lower bandwidth devices.

We now understand why we can't just reduce the segment size blindly. But we do need to deliver smaller chunks of data to client, what if we break this segment in smaller "part"s, each part is not required to have a keyframe, but if it does we can mention it as "INDEPENDENT".

This way client only looks for parts which are "INDEPENDENT" and start decoding from that point, it does not need to wait for complete segment to start playing. This also became possible due to Chunked CMAF and Partial TS. We mentioned about these in previous article, these are container type formats, a way to store video/audio data, these formats getting support chunks/smaller parts allowed us to shrink normal container sizes into n number of chunks. Ideal size of these parts are 300-400ms.

This is the same method used by CL-HLS and LL-HLS both to reduce latency. And the tag used to identify a partial segment is: EXT-X-PART. If we check the spec, we find another interesting attribute BYTERANGE, this is a way to indicate that delivered part will be in this byterange of complete segment.

## Delivery

Okay this seems good, but how does client get these parts? Client will have to fetch manifest to understand how to find them, though as we keep on generating more parts, we keep on changing manifest file really fast and hence we would also have to fetch new changes really quickly. To optimize this step, CL-HLS and LL-HLS took two different routes.

CL-HLS optimized it by pre-announcing segments and when they were fetched, it used HTTP Chunked Transfer Encoding(CTE) to continuously deliver parts.

CTE is a method of sending data in chunks, so instead of letting client know what is the total size of payload, we keep on sending chunks with mentioned size and then when we are done, we send an empty chunk. This is usually employed in cases when size of actual payload is unknown.

One can understand why this is a nice techinque, we are announcing yet to be formed segments, and leverage the network round trip-time to make part and start delivering them continuously using CTE. One point about using CTE is it makes bandwidth estimation harder.

LL-HLS took a different route, in their initial 2019 announcement, it seemed their strategy was to write segments to manifest file if and only if they are generated. Now to make up for low latency, they started pushing segments using HTTP/2 Push when client requests for newer manifest. This saves a round trip-time of client reading the manifest, understanding it and then requesting for segments/parts.

Later, after taking feedback from the community they modified the spec in 2020 to include a prefetch tag which like CL-HLS version allowed to pre-announce segments. So HTTP/2 Push isn't a mandatory requirement anymore.

But client has to still fetch these after getting manifest, no CTE involved. This does make few things more bandwidth consuming. A protocol with all the apple's changes and CTE for continous delivery would be a really good middle-ground and maybe the ultimate goal community was planning for low-latency HLS.

Tag used to pre-announce a segment is EXT-X-PRELOAD-HINT.

## Playlist Delta Updates:

In HLS, another problem we discussed was around big playlists, let's say we have a really long livestream going on, fetching manifest listing segments from the very start till the end would make round-trip super heavy over time. Instead we could get only the updates, deltas from playlist one point in time, this is enabled by playlist delta updates. This is an interesting feature, and hence also ported back to HLS specificiation.

EXT-X-SKIP tag is used as a marker to skip section of manifest. To request for a delta update from server, client uses_HLS_skip=YES|v2 query param.

## Blocking Playlist Updates:

For CDNs, lets say we set cache TTL to 2 seconds, this would mean CDN will not server newer segements produced every 300-400ms till 2 seconds. We may need some kind of cache busting and precise segment/part fetch mechanism, this feature is also provided by LL-HLS in the form of blocking playlist updates. We can tell the server to not break connection and send response until a particular segment/part is ready to be served.

This also helps in client blocking until their set buffer size is not fulfilled at which point they can start decoding and dispalying output instantly.

Blocking is obtained by usage of two query params in sync:
- _HLS_msn=<M>: Do not resolve request until M numbered media sequence number is ready.
- _HLS_part=<N>: Do not resolve request until N part of M segment is ready. This tag needs msn tag to be present.

## Rendition Reports

Another nice feature added by LL-HLS was a faster way to do ABR using rendition reports, these reports contains information like last media sequence number and latest part, etc. A rendition report is needed for each of the defined bitrate individually. EXT-X-RENDITION-REPORT is used to idenitfy these reports.

## Example playlists

 [Apple documentation](https://developer.apple.com/documentation/http-live-streaming/enabling-low-latency-http-live-streaming-hls#Utilize-New-Media-Playlist-Tags-for-Low-Latency-HLS) around LL-HLS provides nice examples for different kinds of playlist requests and their responses.

General low-latency playlist example for request style:
```
GET https://example.com/2M/waitForMSN.php?_HLS_msn=273&_HLS_part=2
```

is:
```
#EXTM3U
#EXT-X-TARGETDURATION:4
#EXT-X-VERSION:6
#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=1.0,CAN-SKIP-UNTIL=12.0
#EXT-X-PART-INF:PART-TARGET=0.33334
#EXT-X-MEDIA-SEQUENCE:266
#EXT-X-PROGRAM-DATE-TIME:2019-02-14T02:13:36.106Z
#EXT-X-MAP:URI="init.mp4"
#EXTINF:4.00008,
fileSequence266.mp4
#EXTINF:4.00008,
fileSequence267.mp4
#EXTINF:4.00008,
fileSequence268.mp4
#EXTINF:4.00008,
fileSequence269.mp4
#EXTINF:4.00008,
fileSequence270.mp4
#EXT-X-PART:DURATION=0.33334,URI="filePart271.0.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.1.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.2.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.3.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.4.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.33334,URI="filePart271.5.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.6.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.7.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.8.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.33334,URI="filePart271.9.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.10.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.11.mp4"
#EXTINF:4.00008,
fileSequence271.mp4
#EXT-X-PROGRAM-DATE-TIME:2019-02-14T02:14:00.106Z
#EXT-X-PART:DURATION=0.33334,URI="filePart272.a.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.b.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.c.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.d.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.e.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.f.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.33334,URI="filePart272.g.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.h.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.i.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.j.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.k.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.l.mp4"
#EXTINF:4.00008,
fileSequence272.mp4
#EXT-X-PART:DURATION=0.33334,URI="filePart273.0.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.33334,URI="filePart273.1.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart273.2.mp4"
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="filePart273.3.mp4"


#EXT-X-RENDITION-REPORT:URI="../1M/waitForMSN.php",LAST-MSN=273,LAST-PART=2
#EXT-X-RENDITION-REPORT:URI="../4M/waitForMSN.php",LAST-MSN=273,LAST-PART=1
```

We can see some familiar tags from HLS post and some new tags which we learned throughout the earlier part of this post. Part duration is defined using tag #EXT-X-PART-INF:PART-TARGET=0.33334. For server control params we see #EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=1.0,CAN-SKIP-UNTIL=12.0, this has a couple of instructions, let's understand them one by one:

- CAN-BLOCK-RELOAD: To block reload until 273 media sequence number and it's 2nd part is not ready.

- Then PART-HOLD-BACK, do not play until 'n', 1.0 here, parts are not available

- And finally CAN-SKIP-UNTIL, server can skip a part of the playlist if it is requested by the client. Value for last tag is in seconds and must be at least six times the target duration, hence we have it as 12 here.

Actual parts are listed using #EXT-X-PART, it has DURATION and URI fields which are pretty self-explanatory. There's also an IDENPENDENT tag at the end of some parts, these represent the I-Frames we talked about earlier and hence can be used as a point from where client can start decoding.

Near the bottom we can see: EXT-X-PRELOAD-HINT, it mentions it's hinting for part using TYPE=PART and then the exact URI to fetch it.

And finally we have rendition reports mentioning last MSN and part index along with URI.

Next they have an example of playlist delta update which is requested using a URL like:
```
GET https://example.com/2M/waitForMSN.php?_HLS_msn=273&_HLS_part=3 &_HLS_skip=YES
```

```
#EXTM3U
# Following the example above, this Playlist is a response to: GET https://example.com/2M/waitForMSN.php?_HLS_msn=273&_HLS_part=3 &_HLS_skip=YES
#EXT-X-TARGETDURATION:4
#EXT-X-VERSION:9
#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=1.0,CAN-SKIP-UNTIL=12.0
#EXT-X-PART-INF:PART-TARGET=0.33334
#EXT-X-MEDIA-SEQUENCE:266
#EXT-X-SKIP:SKIPPED-SEGMENTS=3
#EXTINF:4.00008,
fileSequence269.mp4
#EXTINF:4.00008,
fileSequence270.mp4
#EXT-X-PART:DURATION=0.33334,URI="filePart271.0.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.1.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.2.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.3.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.4.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.33334,URI="filePart271.5.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.6.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.7.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.8.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.33334,URI="filePart271.9.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.10.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart271.11.mp4"
#EXTINF:4.00008,
fileSequence271.mp4
#EXT-X-PROGRAM-DATE-TIME:2019-02-14T02:14:00.106Z
#EXT-X-PART:DURATION=0.33334,URI="filePart272.a.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.b.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.c.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.d.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.e.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.f.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.33334,URI="filePart272.g.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.h.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.i.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.j.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.k.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart272.l.mp4"
#EXTINF:4.00008,
fileSequence272.mp4
#EXT-X-PART:DURATION=0.33334,URI="filePart273.0.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.33334,URI="filePart273.1.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart273.2.mp4"
#EXT-X-PART:DURATION=0.33334,URI="filePart273.3.mp4"
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="filePart273.4.mp4"


#EXT-X-RENDITION-REPORT:URI="../1M/waitForMSN.php",LAST-MSN=273,LAST-PART=3
#EXT-X-RENDITION-REPORT:URI="../4M/waitForMSN.php",LAST-MSN=273,LAST-PART=3
```

We have a new query param of _HLS_SKIP to indicate part of playlist can be skipped. Then we have #EXT-X-SKIP:SKIPPED-SEGMENTS=3 to mention how many segments we have skipped in this playlist update.

Next they have an example of playlist which contains byterange-addressed parts:
```
# In these examples only the end of the Playlist is shown.
# This is Playlist update 1
#EXTINF:4.08,
fs270.mp4
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="20000@0"
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="23000@20000"
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="18000@43000"
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="fs271.mp4",BYTERANGE-START=61000


# This is Playlist update 2
#EXTINF:4.08,
fs270.mp4
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="20000@0"
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="23000@20000"
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="18000@43000"
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="19000@61000"
#EXTINF:4.08,
fs271.mp4
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="fs272.mp4",BYTERANGE-START=0


# This is Playlist update 3
#EXTINF:4.08,
fs270.mp4
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="20000@0"
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="23000@20000"
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="18000@43000"
#EXT-X-PART:DURATION=1.02,URI="fs271.mp4",BYTERANGE="19000@61000"
#EXTINF:4.08,
fs271.mp4
#EXT-X-PART:DURATION=1.02,URI="fs272.mp4",BYTERANGE="21000@0"
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="fs272.mp4",BYTERANGE-START=21000
```

Notice the use of BYTERANGE attribute at the end of each EXT-X-PART tag. Another interesting attribute is BYTERANGE-START in EXT-X-PRELOAD-HINT tag, that is to mark start of byterange of part in segment.

## Conclusion

Need for low-latency solutions arised soon after release of HLS, and we went from no solutions to a couple of competing solutions in no time, it's interesting to watch different approaches and thought process go behind same problem optimization. Today, we have LL-HLS as the leading standard for low latency livestreaming. Amazon/Twitch evolved CL-HLS on their own, and use a proprietary implementation which seems to be performing really well!

Scaling low-latency solutions is definitely hard, and that's why we at Dyte handle that for you, feel home with that sweet DX and leave all the complexity to us, try out our [Livestreaming SDK](https://dyte.io/live-streaming-sdk) today!
