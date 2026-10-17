# Build a media test asset and design its interactive delivery

[Domain index](README.md) · [All tutorials](../README.md)

This tutorial creates or specifies a short synthetic clip, validates a media pipeline, calculates streaming and game-session capacity, and compares desktop/application delivery. All expensive live services remain optional. Use content you created; the exercise does not require licensed films, real users, or public broadcasts.

## 1. Create and inspect a known test clip

If FFmpeg is installed from a trusted distribution, create an eight-second color-pattern clip with a generated tone:

```powershell
ffmpeg -f lavfi -i testsrc=size=640x360:rate=24 -f lavfi -i sine=frequency=440:sample_rate=48000 -t 8 -c:v libx264 -pix_fmt yuv420p -c:a aac -shortest tutorial.mp4
ffprobe -v error -show_entries format=duration:stream=codec_name,width,height,r_frame_rate -of json tutorial.mp4
```

The expected video is approximately eight seconds, 640×360, 24 frames/second, with audio. Encoder availability depends on your FFmpeg build. If `libx264` is unavailable, choose an installed compatible encoder or use a self-recorded short clip; do not download an unknown binary merely to match a command.

Play the file and confirm duration, visible motion, audio, and seek behavior. Keep the metadata output as the baseline. Add a small caption file if learning captions and verify timing rather than assuming a successful encode preserved accessibility.

## 2. Translate the clip into a video-on-demand pipeline

Use the flow private source object → MediaConvert job → private output objects → controlled CloudFront delivery → player. For an optional AWS extension, upload only this tiny clip, create a dedicated least-privilege service role, choose a supported input/output configuration, and submit one job with a deliberately small output ladder.

For example, produce a 360p rendition and a lower-resolution rendition. Inspect codec compatibility, aspect ratio, audio, captions, segment/manifest behavior, and output duration. Verify the player can switch renditions. A completed job status does not establish that the chosen browser/device can decode the result.

Restrict source/output access and use appropriate origin access control for a private S3 REST origin. Decide whether playback requires signed access or application authorization. Publicly accessible origin objects can bypass a protected application UI. Keep DRM and rights-management requirements separate from simple transport encryption.

## 3. Compare live and interactive media roles

MediaConnect handles supported professional transport; MediaLive handles live encoding; MediaPackage handles supported packaging/origin behavior; CloudFront distributes content; IVS provides a managed path for suitable interactive streaming. These roles can be combined but should not be confused.

For a live design, record input redundancy, encoding channels, output formats, latency, regional failure behavior, player compatibility, captions, recording, and operator ownership. Test loss of one input and a malformed stream in a dedicated environment. Do not leave a live channel running while writing a report: idle configuration can still incur charges depending on the service/resource state.

MediaTailor supports ad insertion and channel assembly. Work through a design where the player requests a manifest, the service obtains appropriate ad decisions/content, and playback uses the resulting segments. Define consent, targeting data, fallback when an ad source fails, and playback measurement. A stitched manifest is not proof that advertising rights or measurement requirements are satisfied.

Elastic Transcoder and MediaStore appear in the shutdown register; use current MediaConvert and appropriate origin/storage patterns rather than deploying an old tutorial unchanged.

## 4. Calculate bandwidth and validate assumptions

At an average delivered bitrate of 3 Mbps, 1,000 simultaneous viewers for one hour produce roughly `3,000,000 × 1,000 × 3,600 / 8 = 1.35 × 10^12` bytes, or 1.35 TB decimal, before overhead. Pricing depends on current service/Region/delivery terms; this is a traffic calculation, not a bill.

Do not multiply every viewer by the sum of all adaptive renditions. Usually each viewer consumes a selected rendition over time, while encoding/storage account for multiple outputs. Include startup requests, manifest/segment requests, audio, retries, regional distribution, and origin-cache behavior in a real estimate.

Define customer-quality metrics: playback start success, startup delay, rebuffering ratio, fatal playback errors, and latency for interactive content. An encoder's healthy status cannot substitute for player-side evidence.

## 5. Plan game-server and game-streaming capacity

GameLift Servers addresses supported multiplayer server hosting, while GameLift Streams addresses interactive game/application streaming. Choose based on whether you need a server process coordinating players or remote rendered frames delivered to users.

For a fictional multiplayer design, 1,000 concurrent players with ten players/session require 100 sessions. If a tested instance handles four sessions, the arithmetic baseline is 25 instances before failure/headroom constraints. Benchmark CPU, memory, network, tick rate, and latency with representative game logic; do not infer capacity from player count alone.

Write session lifecycle: matchmaking request, placement, server allocation, player authentication, connection, persistence, termination, and cleanup. Define abandoned-session timeouts, duplicate placement requests, player reconnect, version compatibility, and anti-abuse controls. A fleet scaling out cannot fix a shared database or matchmaking bottleneck.

For streaming, measure controller/input latency and frame delivery separately from ordinary web request latency. Test actual client locations and network conditions. Persist progress in an authoritative store; a terminated transient session should not silently erase accepted progress.

## 6. Treat rendering as a reproducible batch job

For Deadline Cloud, define a tiny render job with input asset versions, output frame range, software/plugin versions, license requirements, and expected output naming. Use a local sample render or generated numbered images as the baseline.

Verify every expected frame exists, dimensions match, and failed frames are visible rather than silently replaced with empty files. Requeue one intentionally failed frame and confirm other successful outputs remain valid. Plan asset transfer, worker sizing, temporary storage, checkpoint/retry behavior, and license concurrency.

Keep job completion and artistic correctness distinct. A process can exit successfully while rendering the wrong camera or missing textures. Include a visual check of selected frames and metadata validation before accepting the output.

## 7. Compare WorkSpaces and application streaming

Create three fictional personas: a persistent office desktop user, a contractor needing one legacy application, and a graphics specialist. Record identity provider/directory requirements, data persistence, latency, peripherals, application licensing, and access duration.

Evaluate supported WorkSpaces desktop options and WorkSpaces Applications, formerly AppStream 2.0. For each, design image creation, patch/update rollout, entitlement, profile persistence, storage, session isolation, and termination behavior. A nonpersistent session can still expose persistent user data through configured storage; specify the boundary explicitly.

Check variant lifecycle before selecting a lab. Secure Browser, Pools, Thin Client, and PCoIP-related offerings have specific notices; do not transfer one feature's status to all WorkSpaces products. Test clipboard, file transfer, printing, local devices, and application uploads according to the required data controls. Remote display alone does not prevent data leaving through authorized applications.

## Verification, troubleshooting, cost, and cleanup

Deliver a playable clip with inspected metadata, one rendition comparison, the 1.35 TB traffic calculation, the 100-session/25-instance baseline, a rendering validation checklist, and the three-persona desktop decision. Label design-only parts accurately.

For broken media, inspect codecs, audio, manifests, CORS, origin access, signed URL expiry, and player logs. For game issues, separate matchmaking, placement, authentication, network, and server tick behavior. For desktop issues, separate directory login, entitlement, image startup, user profile, and network quality.

Optional costs include transcoding, live channels, origin/delivery traffic, IVS, ad insertion, render workers/licenses, game fleets/streams, desktops, directories, and retained storage. Stop live sessions/channels/workers first, then remove dedicated jobs/fleets/images/distributions and stored assets according to dependencies. Do not remove shared directories or licensing infrastructure.

References: [FFmpeg documentation](https://ffmpeg.org/documentation.html), [AWS media services](https://aws.amazon.com/media-services/), [MediaTailor tutorials](https://docs.aws.amazon.com/mediatailor/latest/ug/getting-started.html), [Deadline Cloud](https://docs.aws.amazon.com/deadline-cloud/), [GameLift](https://docs.aws.amazon.com/gamelift/), [WorkSpaces Applications](https://docs.aws.amazon.com/appstream2/latest/developerguide/what-is-appstream.html).
