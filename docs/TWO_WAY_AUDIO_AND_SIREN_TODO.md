# Roadmap & Architecture: Two-Way Audio (Talk) and Siren Integration

This document outlines the context, current state, architectural options, and implementation checklist for supporting **Siren** and **Two-Way Audio (Talk)** for Reolink cameras (such as the Reolink Lumus) through Neolink, Frigate, and Home Assistant.

---

## 1. Context & Objectives

- **Target Hardware**: Mains-powered Reolink Wi-Fi cameras (e.g., Reolink Lumus, E1 series) equipped with a microphone, speaker, spotlight, and siren.
- **Consumer Stack**: 
  - **Frigate / go2rtc**: Used for 24/7 NVR recording, object detection, and live view with two-way audio (microphone button in web UI).
  - **Home Assistant**: Used for smart home dashboards, notifications, and automations (e.g., sounding the siren on intrusion detection, push-to-talk, or text-to-speech announcements).
- **Core Challenge**: Reolink cameras do not expose the siren or microphone over standard RTSP/ONVIF out-of-the-box. Instead, these services operate over Reolink's proprietary **Baichuan protocol** (TCP port 9000) using XML control packets and ADPCM audio streams.

---

## 2. Current Implementation in Neolink

### A. Siren
- **Protocol Layer**: Implemented in `crates/core/src/bc_protocol/siren.rs` (`cam.siren().await`). Sends Baichuan `MSG_ID_AUDIOALARM` to trigger the siren.
- **MQTT Service**: Implemented in `src/mqtt/mod.rs`. Listens for `on` on `neolink/<camera_name>/control/siren`.
- **Home Assistant Discovery**: Supported in `src/mqtt/discovery.rs`. Adding `features = ["siren"]` automatically publishes an MQTT Siren entity for Home Assistant.
- **Status**: **Fully functional.** Only requires running in `mqtt-rtsp` mode with an MQTT broker configured.

### B. Two-Way Audio (Talk)
- **Protocol Layer**: Implemented in `crates/core/src/bc_protocol/talk.rs`:
  - `cam.talk_ability()`: Queries camera audio capabilities (sample rate, block size).
  - `cam.talk(&adpcm, config)`: Sends a one-off audio buffer to the speaker.
  - `cam.talk_stream(rx, config)`: Asynchronously streams continuous ADPCM audio chunks to the camera speaker.
- **CLI Command**: Implemented in `src/talk/mod.rs`:
  ```bash
  neolink talk --config=neolink.toml --adpcm-file=audio.adpcm <camera_name>
  ```
- **RTSP / WebRTC Integration**: **Not yet implemented.** Neolink's GStreamer RTSP server is currently downstream-only (camera → client). It does not expose an RTSP backchannel to receive upstream audio from Frigate/go2rtc.

---

## 3. Two-Way Audio: Integration Approaches

### Option 1: Native RTSP Backchannel (ONVIF Profile T) in Neolink (Recommended for Frigate)
- **Architecture**:
  ```text
  [Frigate / go2rtc Web UI]
         │ (WebRTC Mic Audio)
         ▼
  [go2rtc]
         │ (RTSP Backchannel / RFC 7826 audio)
         ▼
  [Neolink GStreamer RTSP Server]
         │ (GStreamer AppSink / Backchannel Pad)
         ▼
  [Neolink Audio Transcoder] (PCM/G.711/AAC -> ADPCM)
         │
         ▼
  [neolink_core::talk_stream]
         │ (Baichuan MSG_ID_TALK over port 9000)
         ▼
  [Reolink Camera Speaker]
  ```
- **How it works**:
  - GStreamer's RTSP server advertises an ONVIF Profile T backchannel audio track (`a=sendonly` in the SDP offer).
  - When the user presses the microphone button in Frigate / go2rtc, go2rtc sends audio back upstream through the existing RTSP session.
  - Neolink receives the backchannel stream, encodes it to Reolink's expected ADPCM format, and passes it into `cam.talk_stream()`.
- **Pros**:
  - Seamless user experience: The Frigate web UI microphone button works natively with zero extra glue scripts.
  - Automatically handles stream lifetime (talk starts when mic button is pressed, stops on release).
- **Work Required**:
  - Update `src/rtsp/gst/factory.rs` to configure backchannel caps on `NeoMediaFactory`.
  - Connect incoming backchannel buffers to `cam.talk_stream()`.

---

### Option 2: go2rtc `exec` Audio Bridge via CLI
- **Architecture**:
  ```text
  [Frigate UI] ──(WebRTC Mic)──► [go2rtc]
                                     │
                 (exec pipe: PCM)   ▼
                    [ffmpeg -f s16le -i - -f s16le -acodec adpcm_ima_wav -]
                                     │ (ADPCM pipe)
                                     ▼
                    [neolink talk --stdin <camera_name>]
                                     │
                                     ▼
                          [Reolink Camera]
  ```
- **How it works**:
  - In go2rtc configuration, configure the camera's backchannel using an `exec:` source that invokes FFmpeg and `neolink talk`.
- **Pros**:
  - Requires minimal changes to Neolink's core RTSP server.
- **Cons**:
  - Requires FFmpeg installed alongside or in the container.
  - Higher latency and process spawn overhead per talk activation.

---

### Option 3: MQTT / HTTP Audio Endpoint (Recommended for Home Assistant)
- **Architecture**:
  ```text
  [Home Assistant Automation / Dashboard]
         │ (HTTP POST /api/cameras/<name>/talk or MQTT topic)
         ▼
  [Neolink HTTP/MQTT Server]
         │
         ▼
  [neolink_core::talk_stream]
         │
         ▼
  [Reolink Camera Speaker]
  ```
- **How it works**:
  - Expose an endpoint or MQTT topic that accepts a WAV/MP3/ADPCM audio file or stream.
  - Enables Home Assistant to play chime sounds, text-to-speech ("Intruder detected"), or pre-recorded clips over the camera speaker.
- **Pros**:
  - Simple to automate in Home Assistant.
  - Great for doorbell chimes and automated voice warnings.

---

## 4. Siren Integration Setup (When Home Assistant / MQTT is Ready)

When you are ready to configure the siren:

1. **Update `docker-compose.yaml`**:
   Change the run mode from `rtsp` to `mqtt-rtsp`:
   ```yaml
   command: ["neolink", "mqtt-rtsp", "--config", "/etc/neolink.toml"]
   ```

2. **Add MQTT Configuration in `config/config.toml`**:
   ```toml
   bind = "0.0.0.0"
   push_notifications = false

   [mqtt]
   broker_addr = "192.168.2.x" # Your MQTT broker IP
   port = 1883
   credentials = ["mqtt_user", "mqtt_password"]

   [[cameras]]
   name = "front_porch"
   username = "admin"
   password = "your_password"
   address = "192.168.2.103:9000"
   stream = "both"
   push_notifications = false
   # Enable Home Assistant auto-discovery:
   mqtt.discovery.topic = "homeassistant"
   mqtt.discovery.features = ["siren", "floodlight"]
   ```

3. **Home Assistant**:
   - Home Assistant will automatically discover a new `siren.front_porch_siren` entity.
   - You can also manually trigger it by publishing `on` to `neolink/front_porch/control/siren`.

---

## 5. Implementation To-Do Checklist

### Milestone 1: Siren Verification & Documentation
- [x] Document Baichuan audio alarm protocol and MQTT handler.
- [ ] Test `neolink mqtt-rtsp` mode with local Mosquitto broker.
- [ ] Verify Home Assistant MQTT discovery for Siren entity on Reolink Lumus.

### Milestone 2: Two-Way Audio (Talk)
- [ ] **Protocol Verification**:
  - [ ] Test `neolink talk` CLI with a sample 16kHz ADPCM audio file against the Lumus.
  - [ ] Confirm audio quality and speaker volume on hardware.
- [ ] **Input Pipeline**:
  - [ ] Add ADPCM encoder support (e.g., converting incoming PCM 16kHz from clients into Baichuan ADPCM chunks).
- [ ] **Integration Bridge**:
  - [ ] Evaluate Option 1 (GStreamer RTSP backchannel) vs. Option 2 (go2rtc exec pipe).
  - [ ] Implement prototype backchannel or HTTP/MQTT talk endpoint.
  - [ ] Test two-way audio latency and push-to-talk release handling with Frigate UI.
