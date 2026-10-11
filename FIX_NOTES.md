# Neolink Stability & File Descriptor Fixes

This document details the issues resolved in this fork of Neolink, particularly for continuous RTSP streaming to NVR systems like Frigate with mains-powered Reolink cameras (e.g., Reolink Lumus, E1 series).

---

## Table of Contents
1. [Overview](#overview)
2. [Issue 1: File Descriptor Exhaustion ("Too Many Open Files")](#issue-1-file-descriptor-exhaustion-too-many-open-files)
3. [Issue 2: Stream Timestamp Rollover (~71.6 Minutes)](#issue-2-stream-timestamp-rollover-716-minutes)
4. [Issue 3: RTSP Pipeline Churn & Client Socket Leaks](#issue-3-rtsp-pipeline-churn--client-socket-leaks)
5. [Issue 4: Push Notification Storm & Config Ineffectiveness](#issue-4-push-notification-storm--config-ineffectiveness)
6. [Summary of Code Changes](#summary-of-code-changes)
7. [Recommended Configuration for Frigate](#recommended-configuration-for-frigate)
8. [Building and Running](#building-and-running)

---

## Overview

When streaming Reolink cameras to Frigate over RTSP, Neolink containers frequently experienced instability, crashes, or hung streams accompanied by logs such as:

```text
[WARN  neolink::common::pushnoti] Issue connecting to push notifications server: Http(reqwest::Error { kind: Request, url: Url { ... host: Some(Domain("android.clients.google.com")), path: "/checkin" }, source: hyper::Error(Connect, ConnectError("dns error", Os { code: 24, kind: Uncategorized, message: "Too many open files" })) })
```
or continuous repeated errors:
```text
[WARN  neolink::common::pushnoti] Issue connecting to push notifications server: ServerError("PHONE_REGISTRATION_ERROR")
```

The visible notification log was a symptom of two separate underlying defects:
1. An unbounded file descriptor leak in GStreamer frame buffer pools.
2. An obsolete and aggressive push notification registration loop that failed to respect configuration settings.

---

## Issue 1: File Descriptor Exhaustion ("Too Many Open Files")

### Root Cause
In `src/rtsp/factory.rs`, `send_to_appsrc` managed buffer pools using:
```rust
let pool = pools.entry(msg_size).or_insert_with_key(|size| { ... });
```
This keyed a `HashMap<usize, gstreamer::BufferPool>` by the **exact byte length** of each media frame.
- Compressed H.264 video frames constantly fluctuate in size.
- In continuous RTSP streaming at 15–25 fps, thousands of unique frame sizes occur within minutes.
- In GStreamer, each `BufferPool` instance allocates internal synchronization handles (socketpairs and event file descriptors).
- Once the number of open file descriptors hits the process limit (`ulimit -n`, typically 1,024 in containers), the OS denies subsequent descriptor allocations with `EMFILE` (`Os { code: 24, message: "Too many open files" }`).
- Once descriptors are exhausted, GStreamer warns `gst_poll_write_control: assertion 'set != NULL' failed`, media pipelines fail, and any background network activity (such as DNS queries or socket connections) crashes.

### Fix
In `src/rtsp/factory.rs`:
- Bounded buffer pools using power-of-two buckets (`size = data.len().next_power_of_two().max(256)`) up to 1 MiB. This limits the total number of pools across the entire stream to approximately 12.
- Added non-blocking acquisition (`BufferPoolAcquireFlags::DONTWAIT`). If all 32 buffers in a bucket are retained during GStreamer preroll, it wraps the owned frame directly instead of blocking the camera receiving thread.
- Oversized (> 1 MiB) or empty frames are wrapped directly without creating new pools.

---

## Issue 2: Stream Timestamp Rollover (~71.6 Minutes)

### Root Cause
In `src/rtsp/factory.rs`, `vid_ts` and `aud_ts` were tracked as `u32` microseconds:
- $2^{32} \text{ microseconds} \approx 4,294.96 \text{ seconds} \approx 71.58 \text{ minutes}$.
- In release builds, this counter wrapped around to zero after ~71.6 minutes, causing timestamp discontinuity in downstream RTSP consumers (Frigate, VLC, go2rtc) and resulting in frozen or aborted recordings.

### Fix
- Widened `aud_ts` and `vid_ts` from `u32` to `u64`.

---

## Issue 3: RTSP Pipeline Churn & Client Socket Leaks

### Root Cause
In `src/rtsp/gst/factory.rs`:
- `NeoMediaFactory` initialized with `set_shared(false)` and `set_suspend_mode(RTSPSuspendMode::Reset)`.
- Every RTSP client connection (Frigate detect stream, record stream, go2rtc, probes) spun up an independent GStreamer pipeline and Baichuan camera session.
- Disconnecting clients triggered pipeline teardown and reconstruction, accumulating socket file descriptors in `CLOSE_WAIT` and causing resource churn.

### Fix
- Set `factory.set_shared(true)` so all RTSP consumers share a single underlying pipeline and camera session.
- Set `factory.set_suspend_mode(RTSPSuspendMode::None)` to keep the pipeline persistent even during transient client disconnects.

---

## Issue 4: Push Notification Storm & Config Ineffectiveness

### Root Cause
1. **Google Endpoint Deprecation**: Google shut down the legacy GCM/FCM check-in API (`android.clients.google.com/checkin`) that `fcm-push-listener` relied on. Registrations now fail permanently with `PHONE_REGISTRATION_ERROR`.
2. **Missing Global Config**: `push_notifications` was only defined under `CameraConfig` (`[[cameras]]`), and did not exist in the root `Config` struct. If a user placed `push_notifications = false` at the top level of `neolink.toml`, Serde silently ignored it, and every camera fell back to `default_true`.
3. **Missing Aliases**: Common variations like `push_notification` (singular) were not aliased.
4. **Tight Retry Loop**: `PushNotiThread::run` slept for a fixed 3 seconds on failure without exponential backoff.
5. **The Red Herring**: When the GStreamer buffer pool leak exhausted file descriptors, the notification thread's 3-second retry loop could not resolve DNS and logged `Os { code: 24, "Too many open files" }`, misleading users into believing the notification feature caused the FD leak.

### Fix
- In `src/config.rs`: Changed the default for `push_notifications` to `false`.
- Added `push_notifications: Option<bool>` to the root `Config` struct with aliases (`push_notification`, `push`, `push_noti`, `notifications`). Root-level settings automatically propagate to all cameras.
- In `src/common/reactor.rs`: When all cameras have `push_notifications = false`, `PushNotiThread` is **never spawned** and makes zero network requests.
- In `src/common/pushnoti.rs`: Implemented exponential backoff (up to 300 seconds) if push notifications are ever explicitly enabled and fail.

---

## Summary of Code Changes

| File | Change |
|---|---|
| `src/rtsp/factory.rs` | Replaced exact-size buffer pools with power-of-two buckets (PR #423); widened `vid_ts` and `aud_ts` to `u64`; added unit regression tests. |
| `src/rtsp/gst/factory.rs` | Enabled `set_shared(true)` and `set_suspend_mode(None)` (PR #400). |
| `src/config.rs` | Defaulted `push_notifications` to `false`; added global `push_notifications` field and aliases (`push_notification`, `notifications`). |
| `src/common/reactor.rs` | Propagated global `push_notifications` to cameras; prevents starting `PushNotiThread` when disabled. |
| `src/common/pushnoti.rs` | Added exponential backoff (3s to 300s) on connection failures. |
| `sample_config.toml` | Documented `push_notifications = false`. |

---

## Recommended Configuration for Frigate

For continuously powered cameras (e.g., Reolink Lumus, E1 Outdoor) integrated with Frigate:

```toml
bind = "0.0.0.0"
bind_port = 8554

# Push notifications are disabled by default, but can be explicitly disabled globally:
push_notifications = false

[[cameras]]
name = "lumus_driveway"
username = "admin"
password = "your_camera_password"
address = "192.168.1.150:9000"
stream = "both" # Exposes /lumus_driveway/main and /lumus_driveway/sub
```

In your Frigate configuration (`config.yml`):

```yaml
cameras:
  lumus_driveway:
    ffmpeg:
      inputs:
        # Record / live view using the high-resolution Main stream
        - path: rtsp://<neolink-ip>:8554/lumus_driveway/main
          roles:
            - record
        # Object detection using the lower-resolution Sub stream
        - path: rtsp://<neolink-ip>:8554/lumus_driveway/sub
          roles:
            - detect
```

---

## Building and Running

### Build Docker Image Locally
```bash
docker build -t neolink:custom .
```

*Note: The included `Dockerfile` builds using `cargo build --release`, which uses default crate features (`gstreamer`), completely omitting the legacy FCM push notification code from the binary.*

### Run Container
```bash
docker run -d \
  --name neolink \
  --restart unless-stopped \
  -p 8554:8554 \
  -v /path/to/neolink.toml:/etc/neolink.toml:ro \
  neolink:custom
```
