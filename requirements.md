# Comprehensive Evaluation of Open-Source Screen Mirroring, Casting, and Remote Control Systems

Modern mobile and desktop operating systems manage video display streaming, remote input injection, and media routing through disparate architectural layers. Implementing screen mirroring, remote control, and selective content streaming between Android and desktop platforms requires navigating system-level permissions, hardware-accelerated video pipelines, and network transport boundaries. This report evaluates active open-source projects operating across these domains, measures their compliance against complex technical requirements, and analyzes the architectural trade-offs inherent in mobile-to-desktop bridging.

## Comparative Evaluation Matrix

To qualify for evaluation, repositories must be open source and actively maintained with codebase updates within the trailing twelve months leading up to late 2026. The shortlisted projects represent the leading implementations across low-level Android Debug Bridge (ADB) streaming, remote desktop infrastructure, browser-based WebCodecs bridging, and media-layer casting protocols.

### Repository Overview

| Project Name | Repository URL | GitHub Stars | Active Release / Commit Status | License |
| :--- | :--- | :--- | :--- | :--- |
| scrcpy | https://github.com/Genymobile/scrcpy | ~153,000 | v5.0.1 (October 2026) | Apache-2.0 |
| Escrcpy | https://github.com/viarotel-org/escrcpy | ~12,200 | v3.3.1 (October 2026) | Apache-2.0 |
| RustDesk | https://github.com/rustdesk/rustdesk | ~127,000 | Active commits (October 2026) | AGPL-3.0 |
| ScreenStream | https://github.com/dkrivoruchko/ScreenStream | ~2,600 | v4.4.2 (July–October 2026) | MIT |
| FCast | https://github.com/futo-org/fcast | ~685 | v3.0.4 / v3.1.0 (October 2026) | MIT |
| ws-scrcpy-web | https://github.com/bilbospocketses/ws-scrcpy-web | ~75 | v0.5.10 (October 2026) | MIT |

### Feature Compliance Matrix

The following table evaluates each project across the twelve functional domains. Ratings reflect capabilities verified directly through source code, official documentation, or project issue trackers. Capabilities that cannot be corroborated directly from documentation or source repositories are marked as unverified.

| Functional Feature Requirement | scrcpy | Escrcpy | RustDesk | ScreenStream | FCast | ws-scrcpy-web |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Multi-Connection (USB, Wi-Fi, Internet) | Partial (USB, Wi-Fi; Internet via manual tunnel) | Partial (USB, Wi-Fi auto-detect; no Internet relay) | Partial (LAN direct IP, Internet relay; no native USB) | Partial (Local Wi-Fi, Global WebRTC; no native USB) | No (Local Wi-Fi only; no USB or Internet relay) | Partial (USB, local TCP/IP proxy; Internet via reverse proxy) |
| 2. Full Screen Duplication | Yes | Yes | Yes | Yes | Yes | Yes |
| 3. Selective App Sharing | Yes (via --new-display & --start-app) | Partial (inherits CLI backend; GUI focus on mirror) | No (captures full physical display surface) | No (full display stream capture only) | Yes (selective video/audio stream casting) | Partial (inherits scrcpy-server virtual display) |
| 4. Two-Way Laptop Control | Yes (UHID / AOA mouse & keyboard) | Yes (keymapping, touch & mouse macros) | Yes (Accessibility / input service injection) | No (view-only receiver pipeline) | Partial (media playback control only) | Yes (browser touch, keyboard, UHID control) |
| 5. Notification & Privacy Shield | Partial (structural via virtual display; no auto-blur) | No (unverified dynamic redaction) | Partial (approval prompt; OS FLAG_SECURE blacking) | No (unverified dynamic content redaction) | Partial (media streams omit OS UI completely) | No (unverified dynamic content redaction) |
| 6. Secure Connection | Partial (ADB RSA key handshake; no app PIN/QR) | Partial (relies on underlying ADB authorization) | Yes (Ed25519 E2EE, connection passwords) | Yes (WebRTC DTLS/SRTP E2EE; Local PIN) | Partial (unverified transport E2EE; basic pairing) | Partial (local TLS, auth token, opt-in login) |
| 7. Easy Discovery & Setup | Partial (mDNS TCP discovery; no QR client UI) | Yes (LAN device auto-discovery interface) | Yes (LAN discovery, 9-digit ID, address book) | Yes (in-app QR code & direct URL display) | Yes (mDNS discovery, receiver QR scanning) | Yes (browser web portal, service launcher) |
| 8. Orientation & Window Options | Yes (aspect-ratio lock, resizable, fullscreen) | Yes (scaling, orientation flip, embedded mode) | Yes (resizable window, fullscreen desktop view) | Yes (browser viewport, fullscreen, native PiP) | Yes (resizable desktop receiver, fullscreen) | Yes (canvas resizing, fullscreen, web view) |
| 9. Battery & Performance Tuning | Yes (MediaCodec HW encode, --turn-screen-off) | Yes (inherits HW encode & screen-off controls) | Partial (HW encoding; cannot blank screen natively) | Partial (HW RTSP encode; screen must stay on) | Yes (phone screen sleeps during media stream) | Yes (MediaCodec HW encode, screen-off flag) |
| 10. Cross-Platform Laptop Client | Yes (standalone portable binary for Win/Mac/Linux) | Yes (Electron/desktop bundle for Win/Mac/Linux) | Yes (native standalone portable client) | Yes (zero install: any HTML5/WebRTC browser) | Yes (standalone desktop receiver packages) | Yes (zero install: any modern web browser) |
| 11. Quality & Latency Adjustments | Partial (explicit bitrate/fps/size; no auto-adapt) | Partial (preset controls; no dynamic adaptation) | Yes (adaptive bitrate, framerate, quality modes) | Partial (manual RTSP codecs; WebRTC auto-rates) | Partial (codec passthrough; no low-latency mode) | Partial (manual WebCodecs/scrcpy parameters) |
| 12. Multi-Device Operations | Yes (concurrent instances via -s <serial>) | Yes (multi-device panel & group control) | Yes (multiple concurrent remote sessions) | Yes (multiple browser clients per stream) | Partial (one-to-one sender-receiver paradigm) | Yes (multi-device dashboard in browser) |

## Architectural and Technical Deep Dive of Candidate Projects

### Genymobile/scrcpy

The scrcpy project stands as the definitive open-source solution for high-performance, low-latency Android screen mirroring and peripheral control. Rather than running a resident agent service packaged as an Android application package (APK), scrcpy executes by uploading a standalone Java binary (scrcpy-server) to the mobile device via ADB and executing it under the Android shell user context. This design circumvents the strict runtime sandboxing imposed on regular third-party applications, providing direct access to low-level internal system services.

Video capture is executed by interfacing directly with Android's hardware video encoder pipeline through the MediaCodec API. The resulting elementary video stream—encoded using H.264, H.265, or AV1—is multiplexed alongside raw audio and transmitted over an ADB socket tunnel directly to the desktop client application. The desktop client, written in C and utilizing SDL3 along with FFmpeg, decodes the stream using hardware-accelerated rendering surfaces such as Direct3D, VideoToolbox, or VA-API.

Control inputs are injected through Linux User HID (UHID) interfaces or the Android Open Accessory (AOA) protocol, which presents the laptop's physical keyboard and mouse directly to the Linux kernel running beneath Android. This design achieves end-to-end latencies between 35 and 70 milliseconds without requiring administrative root privileges on the device.

### viarotel-org/escrcpy

Escrcpy addresses the usability constraints of scrcpy by wrapping its command-line execution model in an Electron and Vue-based graphical desktop client. Instead of requiring terminal commands and manual configuration flags, Escrcpy provides an integrated graphical dashboard for device orchestration.

The application integrates subnet discovery daemons that actively scan local area networks for exposed wireless ADB ports, eliminating the need to manually execute network pairing commands. It also includes configuration panels for reverse network tethering (gnirehtet), embedded multi-device grid streaming, and extensive keymapping customization.

Recent iterations introduce automation workflows based on the Model Context Protocol (MCP) and visual condition tracking, enabling complex input automation based on graphical changes detected within the mirrored video frame. However, because Escrcpy relies entirely on an embedded or linked scrcpy core, its fundamental capabilities and network boundaries match those of the underlying scrcpy binary.

### rustdesk/rustdesk

RustDesk provides a complete remote desktop infrastructure designed as an open-source alternative to proprietary enterprise platforms such as TeamViewer and AnyDesk. Unlike ADB-based utilities, RustDesk operates on Android as an installed client application that interfaces with the public Android operating system APIs.

The streaming pipeline relies on Android's MediaProjection API to capture visual frames from the primary display, offloading encoding to internal software or hardware video encoders. Remote control is achieved through Android's Accessibility Services, where desktop mouse clicks and keystrokes are received via network packets and translated into synthesized accessibility click, scroll, and text entry gestures.

The primary architectural advantage of RustDesk lies in its networking architecture. It incorporates native public-key cryptography (Ed25519) and symmetric session encryption, combined with rendezvous and relay services (hbbs and hbbr). This design allows devices separated across different networks, NAT boundaries, and firewalls to establish direct peer-to-peer UDP connections or fail over to encrypted TCP relays automatically.

### dkrivoruchko/ScreenStream

ScreenStream is an on-device Android streaming utility designed to eliminate client-side installation requirements on receiving devices. Packaged as a standard APK, the application implements three distinct networking pipelines:

- **Local MJPEG Mode:** An embedded HTTP server processes display frames captured via MediaProjection, compresses them into individual JPEG frames, and serves them over local Wi-Fi to any web browser connecting to the device's IP address.
- **Global WebRTC Mode:** The application coordinates with an open-source signaling server (ScreenStreamWeb) to establish peer-to-peer WebRTC connections over the internet, protected by DTLS/SRTP encryption and an application PIN.
- **RTSP Server/Client Mode:** The device encodes display frames using hardware H.264 or H.265 via MediaCodec and serves the feed via Real-Time Streaming Protocol (RTSP) over TCP/UDP, enabling reception by desktop media players such as VLC or MPV.

ScreenStream operates strictly as a unidirectional screen broadcasting tool. It includes no mechanism for remote input injection, keyboard mapping, or laptop-to-phone hardware control.

### futo-org/fcast

FCast represents an open casting architecture engineered to provide an unencumbered alternative to closed wireless display ecosystems such as Google Cast and Apple AirPlay. Developed by FUTO, the platform is divided into a lightweight control protocol, sender implementations, and high-performance desktop and mobile receivers.

The fundamental design of FCast focuses on media delegation rather than continuous display mirroring. An on-device sender application, such as the Grayjay mobile application, uses the FCast SDK to discover nearby desktop receivers via mDNS or QR code pairing. Once connected, the mobile app passes media metadata, DASH/HLS manifests, or raw video containers to the receiver.

The desktop receiver, written in Rust and backed by GStreamer rendering sinks, manages media retrieval, decoding, and playback directly on the laptop hardware. While FCast includes experimental sender utilities for raw screen duplication, its primary architecture centers on media casting, which allows the mobile device to offload media tasks and preserve battery.

### bilbospocketses/ws-scrcpy-web

The ws-scrcpy-web project represents a modern browser-based implementation descended from the earlier ws-scrcpy codebase. It acts as a lightweight bridging layer between low-level Android system streaming and modern web standards.

The backend service, compiled in TypeScript and running on Node.js, manages ADB socket tunnels connected to physical or network-connected Android targets. It deploys unmodified upstream scrcpy-server binaries to connected mobile devices without requiring Java modifications. Raw H.264/H.265 video feeds generated by the mobile device are packetized and transported over WebSocket connections directly into a web browser.

On the desktop client side, decoding is performed entirely inside the browser using the W3C WebCodecs API, which provides low-latency hardware video decoding without relying on WebAssembly software fallbacks or native desktop application installs. Control inputs captured via browser event listeners are translated into scrcpy binary protocol control packets, granting mouse, touch, and UHID keyboard control inside any modern browser window.

## Domain Analysis Across Core Engineering Vectors

### Connection Transport, Discovery, and Security Models

Screen mirroring and remote control applications rely on fundamentally different transport mechanisms depending on whether they operate over local or wide-area networks:

```
[Local Network Operations]
Wired USB / Local Wi-Fi (ADB Sockets) ──> scrcpy / Escrcpy / ws-scrcpy-web
Local Wi-Fi (HTTP / RTSP / mDNS)     ──> ScreenStream / FCast

[Wide-Area Network Operations]
Public Internet (WebRTC / DTLS / SRTP) ──> ScreenStream Global Mode
Public Internet (NAT Traversal / Relays) ──> RustDesk (hbbs / hbbr)
```

ADB-based architectures (scrcpy, Escrcpy, ws-scrcpy-web) rely on ADB server-client transport. Over physical USB connections, performance is bounded only by the USB controller's throughput and device encoding pipelines, providing ultra-low transmission latency. For wireless operation, the system relies on ADB over TCP/IP (`adb tcpip 5555`), which modern scrcpy versions can discover automatically over local networks via mDNS.

Security within ADB-based systems is handled by an asymmetric RSA key-pair exchange managed by the Android operating system. When an unrecognized host connects, Android issues a system modal dialog prompting the user to authorize the computer's RSA public key fingerprint and optionally save it as a trusted endpoint. However, these systems do not include native transport-layer encryption (TLS) across the local network link; raw frames and control packets travel unencrypted across the local Wi-Fi medium unless wrapped in an external VPN or SSH tunnel.

By contrast, RustDesk and ScreenStream (in Global Mode) are architected specifically to traverse unmanaged networks and the public internet. RustDesk handles connection establishment through signaling servers (hbbs), utilizing UDP hole-punching for direct peer-to-peer links and falling back to encrypted relay servers (hbbr) when symmetrical NAT prevents direct routing. Connections are secured end-to-end using Ed25519 public keys paired with symmetric encryption, while discovery is streamlined via 9-digit numerical device identifiers and PIN-based authorization prompts.

ScreenStream simplifies local setup by hosting an embedded web server that renders a dynamic pairing QR code on the mobile screen, allowing laptops on the same subnet to connect immediately without pre-installed software. In Global WebRTC mode, it relies on DTLS and SRTP encryption standards, routing streams across the internet through WebRTC signaling intermediaries.

### Display Topologies and Selective Application Casting

The technical requirement to isolate a single application (such as casting a video player while interacting privately with a messaging or social media application on the phone) reveals a fundamental architectural divide between full-framebuffer capturing and virtual display composition:

```
[Standard Framebuffer Duplication: RustDesk / ScreenStream]
Android Physical Display 0 ──> Captures ALL pixels, notifications, and UI ──> Laptop Screen

[Virtual Display Pipeline: scrcpy]
Android Virtual Display    ──> Captures ONLY Target App (e.g., VLC)       ──> Laptop Screen
Android Physical Display 0 ──> Retained for private user interaction      ──> Phone Screen
```

Applications built on standard Android user APIs (RustDesk, ScreenStream) rely entirely on the `android.media.projection.MediaProjection` framework. Under this framework, the Android system grants permission to capture the primary display surface (Display 0). Every element rendered to this display—including system status bars, incoming heads-up notifications, application overlays, and personal multi-tasking actions—is captured and streamed to the receiving laptop. It is technically impossible under standard MediaProjection for a user to interact with an unmirrored app on the phone while streaming a different foreground app to a computer.

To bypass this limitation, scrcpy leverages its elevated shell privileges to interface directly with Android's internal DisplayManager service. By invoking the command line parameters `--new-display=[width]x[height]` and `--start-app=[package.name]`, scrcpy instructs Android's SurfaceFlinger and WindowManager to allocate a secondary, virtual display context.

When the target application is launched into this virtual display, its visual surface is rendered independently at the requested resolution and density. The scrcpy capture pipeline attaches to this virtual display surface, streaming only the designated application's output to the laptop window.

Because Android handles the phone's physical screen as Display 0, the user can simultaneously navigate home screens, chat applications, and social media feeds on the physical handset without any of those visual elements appearing on the laptop's display. Virtual display system decorations can be hidden entirely using `--no-vd-system-decorations`, and soft keyboard input can be isolated using `--display-ime-policy=local`.

FCast handles this use case at the media protocol layer. By separating media content from display pixels, an FCast sender extracts the media stream (such as a local file or network video stream) and delegates playback to the laptop receiver. The phone is freed from decoding or displaying the video, allowing the user to multitask or navigate other applications without interrupting the cast.

### Input Injection Mechanics and Control Fidelity

Two-way laptop control requires capturing desktop pointer and keyboard input events and translating them into interactions on the mobile device:

```
[Kernel-Level Input: scrcpy]
Laptop Input ──> SDL3 Capture ──> scrcpy Protocol ──> Linux /dev/uhid ──> Android Input Flinger

[Accessibility-Level Input: RustDesk]
Laptop Input ──> Network Packet ──> Android App ──> Accessibility Service ──> Gesture Dispatch
```

The scrcpy framework uses kernel-level input simulation. When configured with `--keyboard=uhid` and mouse capture flags, scrcpy injects input directly into the Linux kernel via `/dev/uhid` or the USB AOA interface. As a result, Android treats the incoming events as arriving from a physically attached USB keyboard and mouse.

This method ensures accurate character mapping, supports complex key combinations and modifiers (such as Alt+Tab, Ctrl+C, and navigation arrows), and enables smooth mouse tracking without the visual latency associated with simulated touch overlays. Furthermore, scrcpy maps middle mouse clicks to the Android Home key and right mouse clicks to the Back action or display power toggle, allowing full device navigation from the computer. Escrcpy and ws-scrcpy-web inherit this input engine, adding visual keymapping layers that translate desktop keyboard and mouse inputs into precise coordinate taps for mobile gaming and automation macros.

RustDesk approaches input injection through Android’s Accessibility API. The user must grant the RustDesk application comprehensive accessibility permissions in the system settings. Incoming network packets are processed by RustDesk's background service, which translates pointer positions into `AccessibilityService.dispatchGesture()` calls.

While this architecture enables basic clicking, swiping, and text entry across unmodified devices without requiring ADB shell access, it introduces input latency and can encounter edge cases with software keyboards, secure text inputs, and high-frequency gestures. ScreenStream and FCast omit input injection entirely, operating strictly as view-only displays or media receivers.

### Notification Privacy, Content Masking, and Security Flags

Privacy protection during screen sharing involves both system-level security constraints and content filtering:

- **Operating System FLAG_SECURE Protection:** Android provides built-in protection for sensitive data via the `WindowManager.LayoutParams.FLAG_SECURE` flag. Applications that handle sensitive data—such as banking software, password managers, and DRM-protected media players—set this flag on their window surfaces. When an application with `FLAG_SECURE` is active, Android's SurfaceFlinger blanks its surface in all screen capture buffers. This protection is enforced across all capture mechanisms: scrcpy, RustDesk, and ScreenStream all receive a blacked-out surface when secure windows are open, preventing passwords and banking data from leaking into the mirrored stream.
- **Automated Notification Redaction Deficits:** Beyond `FLAG_SECURE`, no active open-source mirroring application includes dynamic, real-time computer vision or optical character recognition (OCR) filters to identify and blur notifications, one-time passwords (OTPs), or incoming messages on the fly.
- **Structural Privacy via Virtual Displays:** While dynamic content filtering is unavailable, scrcpy provides an effective alternative through architectural isolation. By streaming a secondary virtual display (`--new-display`), system heads-up notifications, incoming SMS banners, and phone call alerts continue to route to the primary physical display (Display 0). Because the laptop client only receives frames from the virtual display surface, the user's notifications and private mobile activities remain hidden from the desktop stream.

### Power Management and Performance Tuning

Continuous screen streaming places heavy demands on mobile processors and batteries, making performance and power optimizations critical:

| Optimization Vector | scrcpy | RustDesk | ScreenStream | FCast |
| :--- | :--- | :--- | :--- | :--- |
| Hardware Video Encoding | MediaCodec (H.264, H.265, AV1) | Software / Hardware Codecs | Hardware MediaCodec (RTSP) / WebRTC | N/A (Delegates playback to laptop) |
| Physical Screen State | Can blank display backlight (-S) | Display must remain on | Display must remain on | Display can sleep during media playback |
| Wake-Lock Management | --keep-active / --stay-awake | Internal service wake-locks | Foreground service wake-locks | Background audio/network playback |
| Dynamic Bitrate Adaptation | Fixed CBR set at launch | Dynamic adaptive rate control | WebRTC adaptive rate control | Fixed by source media bitstream |

In standard mirroring workflows, keeping the mobile device's physical screen powered on can cause thermal throttling and battery drain. scrcpy mitigates this with the `--turn-screen-off` (`-S`) parameter, which disables the physical display backlight while keeping the internal graphics compositor and hardware encoder active. On Android 15 and newer, scrcpy manages display power states directly through system shell calls (`cmd display power-off 0`), allowing long streaming sessions without screen burn-in or unnecessary power draw. Furthermore, the `--keep-active` flag prevents the operating system from entering deep sleep states by periodically simulating low-level user activity, maintaining the stream without altering system-wide sleep timeout configurations.

Applications utilizing MediaProjection (ScreenStream, RustDesk) are constrained by Android's power management framework. If the physical screen is powered off or allowed to sleep, Android suspends frame callbacks to the ImageReader or MediaCodec surfaces, interrupting the video stream. Consequently, these applications must keep the physical screen illuminated while streaming, which increases battery consumption.

For media casting, FCast sidesteps this limitation entirely. Because the laptop receiver handles media decoding and playback independently, the phone's hardware encoders remain idle and the physical display can sleep normally during playback, preserving battery.

## Optimal Selection and Dual-Tool Deployment

### The Single Best Standalone Match: scrcpy

When evaluated against the required criteria, scrcpy emerges as the single most capable open-source project. It satisfies more core functional requirements than any alternative, providing:

- Ultra-low latency streaming (35 to 70 ms) via direct hardware-accelerated encoding (H.264, H.265, and AV1).
- Two-way control with kernel-level UHID and AOA mouse and keyboard forwarding.
- Selective application streaming through its virtual display architecture (`--new-display` and `--start-app`), isolating target applications while the user continues to use the phone privately.
- Physical mobile screen blanking via `--turn-screen-off` (`-S`), reducing power consumption and thermal load during extended sessions.
- Portable, lightweight client packages for Windows, macOS, and Linux that run without complex installation procedures.

The primary trade-off with scrcpy lies in its network transport architecture. Because it relies on ADB over USB or local TCP/IP, it does not include built-in NAT traversal or rendezvous servers for connecting across the public internet. Remote internet streaming requires wrapping scrcpy's traffic in a secure tunnel, such as WireGuard, Tailscale, or an SSH tunnel. Users who prefer a graphical interface can use Escrcpy, which bundles the scrcpy engine with an integrated GUI, automated wireless discovery, and visual input mapping.

### The Recommended Dual-Tool Combination

Because no individual open-source project provides both low-level hardware control and zero-configuration public internet routing, combining two specialized tools offers the most complete coverage:

1. **scrcpy (or Escrcpy) for Local High-Performance Streaming and App Isolation:**
   Deployed for local USB and Wi-Fi connections, scrcpy provides virtual display application isolation, physical screen blanking, responsive low-latency peripheral control, and granular encoder tuning.
2. **RustDesk for Remote Internet Control and Traversal:**
   Deployed for remote sessions over the public internet, RustDesk handles wide-area routing through its built-in rendezvous and relay infrastructure (`hbbs` and `hbbr`), secured by Ed25519 asymmetric cryptography and featuring adaptive bitrates for varying network conditions.

## Features Unsupported by Current Open-Source Tooling

Detailed evaluation of the open-source landscape confirms that two requested capabilities are currently unsupported by any single open-source project:

### 1. Automated Dynamic Notification Masking with One-Tap Stream Blurring

No open-source project implements an automated, real-time computer vision or OCR pipeline capable of detecting incoming heads-up notifications, one-time passwords, or sensitive on-screen text fields and dynamically applying blur or redaction filters to the outgoing video stream.

Under Android's security model, content protection is binary: windows marked with `FLAG_SECURE` are blacked out entirely by SurfaceFlinger, while all other visible UI elements are captured as-is.

The closest available architectural alternative is scrcpy's virtual display pipeline (`--new-display`). By rendering the target application to a separate virtual display surface, system notifications, status bars, and third-party popups remain on the primary physical display and are excluded from the stream entirely.

### 2. Unified Multi-Transport Architecture with Integrated Relays

The open-source ecosystem remains split between ADB-based local streaming pipelines (scrcpy, Escrcpy, ws-scrcpy-web) and WebRTC or relay-based remote desktop systems (RustDesk, ScreenStream).

No single project combines raw USB hardware streaming, local mDNS discovery, and integrated STUN/TURN public internet relays with automatic failover under a single client and pairing workflow. Achieving this level of flexibility requires pairing specialized local mirroring utilities with dedicated remote desktop solutions, or routing local streaming tools through an overlay network such as WireGuard or Tailscale.
