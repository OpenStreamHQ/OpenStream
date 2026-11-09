# OpenStream

## The Vision
We are building a modern, completely free-to-try, open-source alternative to legacy P2P streaming tools like *Acestream*, *SopCast*, and *Soda Player*. 

By combining modern Rust headless daemons, Nostr for decentralized stream discovery, Chaumian E-Cash (Cashu) for micro-incentives, and `iroh` (QUIC) for transport, we are creating an unstoppable, buffer-free streaming protocol for both modern PCs and cheap legacy Android TV boxes—with **zero subscriptions, zero accounts, and a self-sustaining P2P economy.**

## Proposed High-Level Architecture

```text
[ Broadcaster: OBS Studio (WHIP / RTMP + Optional L-Bar Sponsor Metadata) ]
           │
           ▼
[ Rust Ingest Node (CMAF LL-HLS Chunker: 500ms .m4s Partials) ]
  ├─► Stream 0: 540p/720p/1080p H.264 (High Profile, Level 4.1 for Android 5 API 21)
  └─► Stream 1: 1080p60 / 4K AV1 (10-bit for Desktop WebCodecs / Modern Hardware)
           │
           ├──► CONTROL PLANE (Nym Mixnet for Broadcaster / OHTTP for Clients)
           │      ├─► Nostr Relays (kind:30311 Live Event + NIP-13 PoW + EPG Health)
           │      └─► OpenStream Cashu Mint (NUT-11 P2PK Watch-Hour Fuel)
           │
           ▼
  DATA PLANE: [ The Swarm (iroh QUIC + 1-Hop Blinded Relay / Dandelion++) ]
  ├─► Manifest Push: iroh-gossip (Plumtree Epidemic Broadcast Tree)
  └─► Media Pull:    iroh-blobs  (BLAKE3 BAO Verified 64KB Bao-Slices + Credit ALPN)
           │
           ▼
[ Rust Core Engine (libnexus_aether.so — Strict 24MB–50MB mmap Ring Buffer) ]
  ├─► Binds: http://127.0.0.1:8888/live.m3u8 (Local)
  └─► Binds: http://0.0.0.0:8888/live.m3u8   (LAN mDNS Remote-Engine Mode)
           │
     ┌─────┴────────────────────────────────────────┐
     ▼                                              ▼
[ Tauri v2 Desktop / Mobile ]         [ Legacy Android 5+ TV (1GB RAM) ]
  • React + WebCodecs / hls.js (4K AV1)      • Native Kotlin + Leanback UI (~18MB RAM)
  • Acts as LAN Super-Node via mDNS          • ExoPlayer (Hardware H.264 + WSOLA)
  • Holds Master Nostr Key & Cashu Seed      • Auto-Offloads P2P to Phone/PC on LAN
```

---

#### I. DEFINITIVE TECHNOLOGY STACK

| Layer | Component | Exact Crate / Library & Hard Constraints |
| --- | --- | --- |
| **Core Daemon** | Rust Headless Engine | Compiled via `cargo-ndk` (`armv7-linux-androideabi`, `aarch64-linux-android`, `x86_64`). Uses `jemalloc` with `dirty_decay_ms: 0` to enforce a hard **50MB RSS ceiling**. |
| **P2P Gossip & Sync** | `iroh` (`v0.32+`) | **`iroh-net`** (QUIC hole-punching + custom 1-hop blinded relays), **`iroh-gossip`** (sub-second chunk announcements), **`iroh-blobs`** (BLAKE3 verified streaming + custom `openstream/media/1` ALPN credit enforcement). |
| **App-Layer Privacy** | Split-Plane Anonymity | **Control Plane:** `nym-sdk` (Rust Mixnet client) for Broadcaster/Mint traffic; OHTTP/TLS for lightweight TV reads. **Data Plane:** 1-Hop Blinded `iroh` Relay + **Dandelion++ peer shuffling** (0% `VpnService` OS popups, 100% live bitrate speed). |
| **Discovery & Pairing** | Nostr (`rust-nostr`) | `kind: 30311` (NIP-53 Live Activities), `kind: 30312` (Swarm Health), **NIP-13** (20-bit PoW anti-spam & 22-bit 10-min trial tickets), **NIP-46 / NIP-44** (6-digit ephemeral pairing + encrypted sub-wallet sync). |
| **Bandwidth Market** | Cashu (`cdk`) | **NUT-11 (P2PK)** atomic **1-satoshi Watch-Hour ticks** (1 sat every 4 mins = 15 sats/hr). Eliminates per-packet Mint HTTP latency and hashchain complexity while guaranteeing zero double-spends. |
| **Legacy TV Client** | Native Kotlin (API 21+) | **AndroidX Leanback** + **Media3 ExoPlayer**. Zero WebView/Flutter/Skia overhead (~18MB UI RAM footprint). Built-in **NsdManager (mDNS)** for LAN offloading. |
| **Desktop / Mobile** | Tauri v2 + React | Native OS WebView + `hls.js` / WebCodecs for hardware 4K AV1 decoding. Doubles as a **LAN Remote Super-Node** for weak TVs in the house. |

---

#### II. HYBRID CORE SUBSYSTEMS (MEMORY & STREAMING PHYSICS)

1. **Zero-Disk Static Slab Ring Buffer (`bytes::Bytes` + `axum`):**
   * The Rust daemon exposes `http://0.0.0.0:8888`. Instead of standard HLS (which lags by 12–18s), it serves **LL-HLS (Low-Latency HLS)** with **500ms CMAF `.m4s` partial segments** and Blocking Playlist Reload (`_HLS_msn` & `_HLS_part`).
   * The RAM slab is pre-allocated at startup (**50MB** on Desktop/Mobile; **24MB** in standalone Low-Spec TV Mode to leave 26MB headroom for QUIC/Rust runtime inside the 50MB RSS limit):
     $$\text{Desktop Buffer} = 100 \times 512\text{ KB Slabs} = 50\text{ MB} \quad \big| \quad \text{TV Buffer} = 48 \times 512\text{ KB Slabs} = 24\text{ MB}$$
   * Oldest slabs are overwritten cyclically. **Zero bytes** are ever written to the Android TV's eMMC flash storage.
   * **Zero-Buffer Adaptive Bitrate:** If a viewer's ISP download speed drops, the daemon seamlessly requests **Stream 0 (540p/720p H.264)** `.m4s` partials on the next 500ms boundary without ever showing a spinning buffer wheel.

2. **BLAKE3 Verified BAO Slicing (Anti-Poisoning):**
   * A malicious peer cannot inject corrupted video frames to crash ExoPlayer. Every 500ms `.m4s` chunk is verified on-the-fly in 64KB slices against the broadcaster's signed BLAKE3 Merkle root via `iroh-blobs` *before* being written into the RAM ring buffer.

3. **Automatic LAN Offloading ("The Beefy Phone / PC Bridge"):**
   * When the Kotlin TV app boots on a weak 1GB TV, it listens for **1,500ms** via Android `NsdManager` for `_nexus-aether._udp.local`.
   * **If a Paired Phone or PC Super-Node is on the Wi-Fi:** The TV does **not** start its local `libnexus_aether.so` P2P daemon. Instead, it sends the selected Nostr stream ID to the phone/PC over LAN and streams `http://PHONE_IP:8888/live/stream0.m3u8` (using `<15MB` RAM on the TV and granting **100% Free Unlimited Access**).
   * **If no Phone/PC is present:** The TV spawns `libnexus_aether.so` locally via JNI in **Low-Spec Mode** (24MB ring buffer, capped at **25 concurrent QUIC peers**, Stream 0 H.264) and runs on either the **10-Minute Free Trial** or **Prepaid Watch-Hours**.

---

#### III. REPUTATION & QUALITY ENGINE

To prevent router NAT exhaustion at boot, the UI renders in `<200ms` using signed `kind: 30312` telemetry from **EPG Probe Nodes**, while the local Rust engine runs live verification in the background on highlighted/active streams using a `0` to `100` score:

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

#### IV. CRITICAL BUSINESS LOGIC, INCENTIVES & WORKFLOWS

##### **Workflow 1: The Binary Economy (Super-Node vs. Watch-Hours + 10-Min Trial)**
Every peer in the `iroh` swarm enforces a custom ALPN credit ledger (`openstream/media/1`) per `NodeId`:
$$\text{Credit}_{\text{peer}} = (\text{Bytes Uploaded} \times 1.2) - \text{Bytes Downloaded} + \text{Bytes Paid via Cashu}$$

* **State 1 — 10-Minute Swarm-Enforced Free Trial (APK-Crack Proof):**
  * New users with no LAN Super-Node and 0 Watch-Hours get **10 minutes of instant, full-quality playback** to build trust.
  * To prevent modded APKs from bypassing the timer, the 10-minute limit is enforced by **remote Super-Seeders, not the local UI**. The client mines a **22-bit NIP-13 PoW Trial Ticket** at channel load. Super-Seeders honor this ticket for **600 seconds (or 150MB)** per `NodeId` and `/24` IP subnet (tracked in a rolling 2-hour Bloom filter).
  * At `10:01`, the swarm drops the connection (`402 Payment Required`). Even if a user wipes app data every 10 minutes, the swarm blocks repeat free trials on that IP, and quitting a live match 12 times is unbearable friction.
* **State 2 — Option A: Free Forever via Verified PC/Phone Super-Node:**
  * At the 10-minute cutoff, the user can pair a PC or Phone on their Wi-Fi. The PC/Phone runs a 5-second `iroh-net` NAT hole-punch and upload test ($\ge 5\text{ Mbps}$).
  * If it passes and seeds to the swarm ($\text{Ratio} \ge 1.0\times$), the PC/Phone acts as a Super-Node and every TV/device in the house streams **100% free forever**. (If their ISP blocks P2P via strict CGNAT, the app explains why and prompts Option B).
* **State 3 — Option B: Prepaid "Watch-Hour" Fuel Tank (~1.5¢ / Hour):**
  * Users behind strict CGNAT (or who only own a TV) scan a Lightning QR code to buy **Watch-Hours** (e.g., **$2.00 = ~130 Watch Hours**, stored as atomic **1-satoshi Cashu tokens** that only drain while actively watching).
  * **The 4-Minute Atomic Tick:** Every **4 minutes** (15 sats / Watch Hour), the Rust daemon locks a **single 1-satoshi token** (`NUT-11` P2PK) in a QUIC control frame:
    * **60% of Ticks (9 sats/hr) $\rightarrow$ Paid to the Super-Seeder Peers** supplying the stream.
    * **40% of Ticks (6 sats/hr) $\rightarrow$ Paid directly to the Broadcaster's Nostr PubKey.**
* **State 4 — Broadcaster & Developer Monetization (Fork-Proof):**
  * **Broadcasters** earn 40% of all Watch-Hour ticks, Root-Slice arbitrage from commercial seeders, and can inject **Unblockable L-Bar Sponsor Banners** (cryptographically bound to the BLAKE3 Merkle tree of the video keyframe so ad-blockers cannot strip them).
  * **Developers** charge **0% in the open-source client code** (making code forks pointless) and collect a **5%–8% Mint Melt Fee** when Broadcasters and commercial Super-Seeders cash out their accumulated Cashu tokens to Lightning via the default OpenStream Mint.

##### **Workflow 2: Dynamic Smooth Catch-Up (WSOLA Pitch-Locked)**
To keep live sports synchronized across all viewers while absorbing P2P jitter:
* **$\Delta t < 3.0\text{s}$ from Live Edge (Normal):** Playback rate = $1.00\times$.
* **$3.0\text{s} \le \Delta t \le 12.0\text{s}$ (Elastic WSOLA Catch-Up):** Playback scales smoothly via ExoPlayer `PlaybackParameters(speed, 1.0f)` (which engages native Time-Domain WSOLA pitch preservation so commentators never sound high-pitched):
  $$\text{Rate} = 1.0 + \min\left(0.20, \frac{\Delta t - 3.0}{60}\right)$$
* **$\Delta t > 12.0\text{s}$ (Hard Live-Edge Snap):** Network handoff occurred. The Rust daemon flushes stale slabs in the ring buffer, pulls the latest `iroh-gossip` keyframe (`IDR` frame) slab, and issues an instant discontinuity seek to the live edge.

##### **Workflow 3: Frictionless 6-Digit Nostr Pairing + Disposable Sub-Wallet**
1. The weak Android TV opens the Kotlin app, generates an **ephemeral `secp256k1` keypair**, and displays a **6-digit code** (e.g., `849-201`) alongside a QR code.
2. On your **Beefy Phone** or **Desktop**, you scan the QR or enter `849-201`.
3. Over Nostr (**NIP-46 / NIP-44 encrypted DM**), the phone pushes two things to the TV:
   * A **Sub-Wallet Cashu Allowance** (e.g., 200 sats max in disposable 1-sat Watch-Hour tokens—**never exposing your master Nostr key or Cashu seed phrase to a cheap TV**).
   * The Phone/PC's **Local LAN IP + Port (`192.168.1.x:8888`)** so the TV can immediately switch into **Zero-Load Remote Engine Mode** whenever the Super-Node is home.

---

#### V. ZERO-INFRASTRUCTURE DISTRIBUTION

* **Self-Bootstrapping Swarm:** The app requires zero central API servers. Upon launch, `rust-nostr` queries a **hardcoded fallback list of 15 censorship-resistant Nostr relays** (plus user-configured relays) for `kind: 30311` streams and `kind: 30312` EPG health attestations.
* **Immortal Web/Desktop Distribution:** The Tauri/PWA web frontend compiles into a **single self-contained `index.html` + `.wasm` bundle** hosted on **IPFS, Blossom (Nostr media servers by SHA-256 hash), and Arweave**.
* **Signed Binary Updates over `iroh`:** App updates (`.apk` for Android TV and `.AppImage`/`.msi` for Desktop) are distributed directly over the **`iroh-blobs` P2P swarm**, verified against the core developers' threshold Nostr Schnorr signatures (FROST)—meaning even if GitHub, Google Play, and every domain registrar take down the project, existing TVs and PCs continue discovering streams and updating over the swarm.

### Next Steps
1. 🔔 [**Subscribe to our Launch Announcement**](https://github.com/OpenStreamHQ/OpenStream/discussions/3) to get pinged the moment we drop the first usable build.
2. 💬 **Have an idea? [Start a Discussion](https://github.com/OpenStreamHQ/OpenStream/discussions).** 
