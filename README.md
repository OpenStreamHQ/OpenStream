# OpenStream

## The Vision
We are building a modern, completely free, open-source alternative to legacy P2P streaming tools like *Acestream*, *SopCast*, and *Soda Player*. 

By combining modern Rust headless daemons, Nostr for decentralized stream discovery, Chaumian E-Cash (Cashu) for micro-incentives, and `iroh` (QUIC) for transport, we are creating an unstoppable, buffer-free streaming protocol for both modern PCs and cheap legacy Android TV boxes.

## Proposed High-Level Architecture

```text
[ Broadcaster: OBS Studio (WHIP / RTMP) ]
           │
           ▼
[ Rust Ingest Node (CMAF LL-HLS Chunker: 500ms .m4s Partials) ]
  ├─► Stream 0: 1080p H.264 (High Profile, Level 4.1 for Android 5 API 21)
  └─► Stream 1: 4K AV1 (10-bit for Desktop WebCodecs / Modern Hardware)
           │
           ├──► CONTROL PLANE (Nym Mixnet / SOCKS5 App-Layer Anonymity)
           │      ├─► Nostr Relays (kind:30311 Live Event + NIP-13 PoW)
           │      └─► Cashu Mint (NUT-11 P2PK Bandwidth Escrow)
           │
           ▼
  DATA PLANE: [ The Swarm (iroh QUIC + 1-Hop Blinded Relay / Dandelion++) ]
  ├─► Manifest Push: iroh-gossip (Plumtree Epidemic Broadcast Tree)
  └─► Media Pull:    iroh-blobs  (BLAKE3 BAO Verified 64KB Bao-Slices)
           │
           ▼
[ Rust Core Engine (libnexus_aether.so — Strict 50MB mmap Ring Buffer) ]
  ├─► Binds: http://127.0.0.1:8888/live.m3u8 (Local)
  └─► Binds: http://0.0.0.0:8888/live.m3u8   (LAN mDNS Remote-Engine Mode)
           │
     ┌─────┴────────────────────────────────────────┐
     ▼                                              ▼
[ Tauri v2 Desktop / Mobile ]         [ Legacy Android 5+ TV (1GB RAM) ]
  • React + WebCodecs / hls.js (4K AV1)      • Native Kotlin + Leanback UI
  • Acts as LAN Super-Node via mDNS          • ExoPlayer (Hardware H.264 + WSOLA)
  • Holds Master Nostr Key & Cashu Seed      • Auto-Offloads P2P to Phone on LAN

```

---

#### I. DEFINITIVE TECHNOLOGY STACK

| Layer | Component | Exact Crate / Library & Hard Constraints |
| --- | --- | --- |
| **Core Daemon** | Rust Headless Engine | Compiled via `cargo-ndk` (`armv7-linux-androideabi`, `aarch64-linux-android`, `x86_64`). Uses `jemalloc` with `dirty_decay_ms: 0` to enforce a hard **50MB RSS ceiling**. |
| **P2P Gossip & Sync** | `iroh` (`v0.32+`) | **`iroh-net`** (QUIC hole-punching + custom 1-hop blinded relays), **`iroh-gossip`** (sub-second chunk announcements), **`iroh-blobs`** (BLAKE3 verified streaming). |
| **App-Layer Privacy** | Split-Plane Anonymity | **Control Plane:** `nym-sdk` (Rust Mixnet client) for Nostr/Cashu traffic. **Data Plane:** 1-Hop Blinded `iroh` Relay + Dandelion++ peer shuffling (0% `VpnService` OS popups, 100% live bitrate speed). |
| **Discovery & Pairing** | Nostr (`rust-nostr`) | `kind: 30311` (NIP-53 Live Activities), **NIP-13** (20-bit PoW anti-spam), **NIP-46 / NIP-44** (6-digit ephemeral pairing + encrypted wallet sync). |
| **Bandwidth Market** | Cashu (`cdk`) | **NUT-11 (P2PK)** + **NUT-14 (HTLC)** batched session tickets. Eliminates per-packet Mint HTTP latency while guaranteeing zero double-spends. |
| **Legacy TV Client** | Native Kotlin (API 21+) | **AndroidX Leanback** + **Media3 ExoPlayer**. Zero WebView/Flutter/Skia overhead (~18MB UI RAM footprint). Built-in **NsdManager (mDNS)** for LAN offloading. |
| **Desktop / Mobile** | Tauri v2 + React | Native OS WebView + `hls.js` / WebCodecs for hardware 4K AV1 decoding. Doubles as a **LAN Remote Engine** for weak TVs in the house. |

---

#### II. HYBRID CORE SUBSYSTEMS (MEMORY & STREAMING PHYSICS)

1. **Zero-Disk 50MB Slab Ring Buffer (`bytes::Bytes` + `axum`):**
* The Rust daemon exposes `[http://0.0.0.0:8888](http://0.0.0.0:8888)`. Instead of standard HLS (which lags by 12–18s), it serves **LL-HLS (Low-Latency HLS)** with **500ms CMAF `.m4s` partial segments** and Blocking Playlist Reload (`_HLS_msn` & `_HLS_part`).
* The 50MB RAM slab is pre-allocated at startup:

$$\text{Buffer Capacity} = 100 \times 512\text{ KB Slabs} = 50\text{ MB (Strict Static Allocation)}$$


* Oldest slabs are overwritten cyclically. **Zero bytes** are ever written to the Android TV's eMMC flash storage.


2. **BLAKE3 Verified BAO Slicing (Anti-Poisoning):**
* A malicious peer cannot inject corrupted video frames to crash ExoPlayer. Every 500ms `.m4s` chunk is verified on-the-fly in 64KB slices against the broadcaster's signed BLAKE3 Merkle root via `iroh-blobs` *before* being written into the 50MB RAM ring buffer.


3. **Automatic LAN Offloading ("The Beefy Phone Bridge"):**
* When the Kotlin TV app boots on a weak 1GB TV, it listens for 1,500ms via Android `NsdManager` for `_nexus-aether._udp.local`.
* **If your Beefy Phone or PC is on the Wi-Fi:** The TV does **not** start its local `libnexus_aether.so` P2P daemon. Instead, it sends the selected Nostr stream ID to the phone/PC over LAN and streams `http://PHONE_IP:8888/live/stream0.m3u8` (using `<15MB` RAM on the TV).
* **If no Phone/PC is present:** The TV spawns `libnexus_aether.so` locally via JNI in Low-Spec Mode (capped at 25 concurrent QUIC peers, Stream 0 1080p/720p H.264).



---

#### III. REPUTATION & QUALITY ENGINE

Before rendering channels in the UI, the Rust engine scores every `kind: 30311` Nostr broadcast event from `0` to `100` using passive telemetry and first-second verification:

$$S_{\text{total}} = 0.20(S_{\text{PoW}}) + 0.40(S_{\text{Cadence}}) + 0.40(S_{\text{Swarm}})$$

* **1. Anti-Sybil & PoW ($S_{\text{PoW}}$ — 20%):**
* Event must carry a valid **NIP-13 Proof-of-Work $\ge 20\text{ bits}$** (takes ~1 second for a real broadcaster to mine, expensive for spam bots) OR a valid **NIP-05 / Cashu Stake Bond**.


* **2. Cadence Continuity ($S_{\text{Cadence}}$ — 40%):**
* Broadcaster signs `iroh-gossip` chunk headers with monotonic timestamps. The Rust engine calculates the rolling standard deviation $\sigma_{\Delta t}$ of the last 10 chunk arrivals (expected $\mu = 500\text{ms}$):

$$S_{\text{Cadence}} = \max\left(0, 100 - \left(\frac{\sigma_{\Delta t}}{2.5\text{ms}}\right)\right)$$


* If $\sigma_{\Delta t} > 100\text{ms}$ (broadcaster's CPU/GPU is dropping frames), score drops below 60 and the stream is hidden or flagged.


* **3. Swarm Health & Tree Depth ($S_{\text{Swarm}}$ — 40%):**
* Evaluates the active `iroh-gossip` Plumtree peer count ($N_{\text{peers}} \ge 3$) and average hop distance from the origin seeder ($H \le 3$). Streams scoring $S_{\text{total}} < 60$ are filtered out by default.



---

#### IV. CRITICAL BUSINESS LOGIC & WORKFLOWS

##### **Workflow 1: Tit-for-Tat + Zero-Latency Cashu Buffer-Rescue**

* **State 1 — Barter Mode (Free):** While the node maintains an Upload/Download ratio $R \ge 0.8$ (or is within its initial 30-second grace window), standard `iroh` tit-for-tat reciprocation keeps the stream 100% free.
* **State 2 — Asymmetric / CGNAT Rescue Trigger:** If the user is behind symmetric CGNAT (cannot seed) and the local RAM buffer drops below **6.0 seconds** ($\le 12$ CMAF partials), the engine triggers **Rescue Mode**.
* **State 3 — Batched Session Ticket + Micro-Tick Unchoking:**
1. Instead of sending a slow Mint-verified Cashu token on every UDP packet, the Rust daemon grabs a **10-satoshi ($0.01) Cashu token** from its local wallet, locks it to a Super-Seeder's Nostr PubKey (**NUT-11 P2PK**), and sends it in a single QUIC control frame.
2. The Super-Seeder immediately unchokes a dedicated high-priority QUIC stream for **15 minutes** (or 150MB of video) and verifies/swaps the 10-sat token asynchronously with the Cashu Mint in the background (over Nym).
3. Each subsequent 500ms chunk request includes a lightweight 8-byte **SHA-256 Hashchain Preimage** ($H^n(s)$), proving part of the prepaid 10-sat credit in **0.01 milliseconds** with zero Mint roundtrips.



##### **Workflow 2: Dynamic Smooth Catch-Up (WSOLA Pitch-Locked)**

To keep live sports synchronized across all viewers while absorbing P2P jitter:

* **$\Delta t < 3.0\text{s}$ from Live Edge (Normal):** Playback rate = $1.00\times$.
* **$3.0\text{s} \le \Delta t \le 12.0\text{s}$ (Elastic WSOLA Catch-Up):** Playback scales smoothly via ExoPlayer `PlaybackParameters(speed, 1.0f)` (which engages native Time-Domain WSOLA pitch preservation so commentators never sound high-pitched):

$$\text{Rate} = 1.0 + \min\left(0.20, \frac{\Delta t - 3.0}{60}\right)$$


* **$\Delta t > 12.0\text{s}$ (Hard Live-Edge Snap):** Network handoff occurred. The Rust daemon flushes stale slabs in the 50MB ring buffer, pulls the latest `iroh-gossip` keyframe (`IDR` frame) slab, and issues an instant discontinuity seek to the live edge.

##### **Workflow 3: Frictionless 6-Digit Nostr Pairing + LAN Discovery**

1. The weak Android TV opens the Kotlin app, generates an ephemeral `secp256k1` keypair, and displays a **6-digit code** (e.g., `849-201`) alongside a QR code.
2. On your **Beefy Phone** or **Desktop**, you scan the QR or enter `849-201`.
3. Over Nostr (**NIP-44 encrypted DM**), the phone pushes two things to the TV:
* A **Sub-Wallet Cashu Allowance** (e.g., 200 sats max in disposable tokens—never exposing your master seed phrase to a cheap TV).
* The Phone's **Local LAN IP + Port (`192.168.1.x:8888`)** so the TV can immediately switch into **Zero-Load Remote Engine Mode** whenever the phone is home.



---

#### V. ZERO-INFRASTRUCTURE DISTRIBUTION

* **Self-Bootstrapping Swarm:** The app requires zero central API servers. Upon launch, `rust-nostr` queries a hardcoded fallback list of 15 censorship-resistant Nostr relays (plus user-configured relays) for `kind: 30311` streams.
* **Immortal Web/Desktop Distribution:** The Tauri/PWA web frontend compiles into a **single self-contained `index.html` + `.wasm` bundle** hosted on **IPFS, Blossom (Nostr media servers by SHA-256 hash), and Arweave**.
* **Signed Binary Updates over `iroh`:** App updates (`.apk` for Android TV and `.AppImage`/`.msi` for Desktop) are distributed directly over the **`iroh-blobs` P2P swarm**, verified against the core developers' threshold Nostr Schnorr signatures (FROST)—meaning even if GitHub, Google Play, and every domain registrar take down the project, existing TVs and PCs continue discovering streams and updating over the swarm.

### Next Steps
1. ⭐ **Star the repo!**  Getting enough stars unlocks free GitHub Copilot tiers for maintainers.
2. 🔔 [**Subscribe to our Launch Announcement**](https://github.com/OpenStreamHQ/OpenStream/discussions/3) to get pinged the moment we drop the first usable build.
3. 💬 **Have an idea? [Start a Discussion](https://github.com/OpenStreamHQ/OpenStream/discussions).** 
