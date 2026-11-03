# OpenStream

> **A zero-infrastructure, censorship-resistant, anonymous P2P live-streaming platform.**


## The Vision
We are building a modern, completely free, open-source alternative to legacy P2P streaming tools like *Acestream*, *SopCast*, and *Soda Player*. 

By combining modern Rust headless daemons, Nostr for decentralized stream discovery, Chaumian E-Cash (Cashu) for micro-incentives, and `iroh` (QUIC) for transport, we are creating an unstoppable, buffer-free streaming protocol for both modern PCs and cheap legacy Android TV boxes.

## Proposed High-Level Architecture

We are decoupling the heavy P2P networking from the UI. A lightweight Rust daemon handles the swarm, routing, and discovery, serving standard video to "dumb" clients.

```text
[ Broadcaster (OBS H.264/AV1) ]
           │
           ▼
  [ The Swarm (iroh QUIC P2P) ] ◄── (App-Layer Onion Routing: Nym/i2pd)
           │
           ▼
[ Rust Core Engine (Local Daemon) ] ◄── (Nostr Discovery / Cashu E-Cash)
           │
           ├─► Exposes http://127.0.0.1:8080/live.m3u8
           │
   ┌───────┴─────────────┐
   ▼                     ▼
[ Tauri PC App ]  [ Android TV App ]
 (React / 4K AV1)      (Kotlin Leanback / H.264)
```

## Tech Stack Highlights
- **Core Engine:** Rust (Compiles to `x86_64`, `armv7`, `aarch64`, Wasm).
- **P2P Transport:** `iroh` (Ultra-lightweight QUIC).
- **Privacy Routing:** Embedded Nym Mixnet or C++ `i2pd` bindings. (Application-layer only to avoid VPN permission prompts on Androids).
- **Discovery:** Decentralized indexing via `rust-nostr`.
- **Micro-Payments:** Cashu (`cdk`) for off-chain micro-settlement.
- **Legacy UI:** Native Kotlin + AndroidX Leanback (Targets 1GB RAM Android 5 API 21).
- **Modern UI:** Tauri v2 + React (Hardware AV1 decoding via `hls.js`).

### Planned Folder Structure

```text
openstream/
├── core/                # The Rust headless daemon (P2P, Nostr, Cashu)
├── clients/
│   ├── desktop-web/     # Tauri + React + WebCodecs frontend
│   └── android-tv/      # Native Kotlin Leanback app (ExoPlayer)
├── docs/                # Architecture blueprints & API specs
└── README.md
```

## 🤝 Calling all "Vibe Coders"
We are building this using modern AI-assisted workflows. If you are a "vibe coder" who loves gluing together complex protocols rapidly using AI tools, we want you! 

### Next Steps
1. ⭐ **Star the repo!**  Getting enough stars unlocks free GitHub Copilot tiers for maintainer.
2. 🔔 [**Subscribe to our Launch Announcement**](https://github.com/OpenStreamHQ/OpenStream/discussions/3) to get pinged the moment we drop the first usable build.
3. 💬 **Have an idea? [Start a Discussion](https://github.com/OpenStreamHQ/OpenStream/discussions).** 
