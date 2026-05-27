🎬 BLACKBOX LOGGER
Comprehensive Engineering Design Document
Version 1.0 | Privacy-First Local "Flight Recorder" for Your PC
TABLE OF CONTENTS

    Project Overview & Vision
    Goals, Non-Goals & Constraints
    System Architecture
    Module Breakdown
        4.1 Screen Capture Engine
        4.2 Input Logger
        4.3 Window & App Tracker
        4.4 Clipboard Monitor
        4.5 Video Encoder & Segment Manager
        4.6 Cryptography & Key Manager
        4.7 OCR Engine & FTS Indexer
        4.8 Search Engine
        4.9 Replay & Export Engine
        4.10 Tauri IPC Command Layer
        4.11 Dashboard UI (Svelte)
        4.12 System Integration (Tray, Autostart, Permissions)
    Data Models & Schemas
    API Specifications (Tauri Commands & Events)
    Directory Structure
    Configuration System
    Cryptography Deep Dive
    Segment File Format (.bbv) Specification
    OCR Pipeline Deep Dive
    Search System Deep Dive
    Replay Engine Deep Dive
    Privacy & Redaction Module
    Security Model & Threat Boundaries
    Storage & Persistence
    Logging, Observability & Debugging
    Testing Strategy
    Build, Packaging & Installation
    Platform Support Matrix
    Performance Targets & Benchmarks
    Error Handling Strategy
    Dependency Registry
    Milestone & Phased Rollout Plan
    Open Questions & Future Work

1. Project Overview & Vision
1.1 What is BlackBox Logger?

BlackBox Logger is a privacy-first, fully local, always-on "flight recorder" for your computer. Inspired by the aviation black box concept, it runs silently in the background, maintaining a rolling encrypted buffer of your screen activity, active application context, input events, and clipboard — then lets you rewind, replay, and search everything that happened, with zero data ever leaving your machine.

It is the deliberate opposite of Microsoft Recall: open-source, cross-platform, cryptographically private, user-controlled, and architecturally incapable of phoning home.
1.2 The Problem Being Solved

Knowledge workers, developers, researchers, and power users routinely face situations where they wish they could "rewind time":

    A crash destroyed unsaved work — what was I typing?
    A bug manifested once and never again — what exactly happened?
    A colleague asked what steps were taken — I can't remember the exact sequence
    A terminal command produced unexpected output — what did I accidentally run?
    An important file was overwritten — what was in it before?

Today, there is no reliable, privacy-respecting answer. Screen recorders require manual start/stop, are not searchable, and store unencrypted video. System logs don't capture screen state. Browser history doesn't capture desktop activity.

BlackBox Logger solves this with an always-on, always-encrypted, fully-local ring buffer that turns your computer's activity history into a searchable, rewindable personal archive.
1.3 Design Philosophy
Principle	Description
Local-first	All capture, storage, AI inference, and search happens on-device. No cloud dependency.
Encrypted at rest	Every segment file, every thumbnail, every OCR result is encrypted with per-artifact keys wrapped by a user-controlled master key.
User-controlled ring buffer	A hard ceiling on disk usage and time retention. Old data is pruned automatically. Nothing is kept forever unless explicitly saved.
Searchable memory	OCR + FTS5 transforms a video archive into a queryable knowledge base.
Zero-network capture path	The capture, encode, and storage pipeline has no network code whatsoever.
Graceful degradation	Errors in OCR, encryption, or indexing never crash capture.
Privacy by default	Sensitive contexts (password managers, private browsing, configurable app list) are never captured.
2. Goals, Non-Goals & Constraints
2.1 Goals (In Scope)

    Always-on rolling screen capture (H.265, 5 FPS default, 720p default)
    Multi-monitor support with per-monitor capture streams
    Window/application focus tracking attached to every point in time
    Input event capture (keyboard shortcuts, mouse clicks/scrolls) with aggressive redaction defaults
    Clipboard change monitoring with deduplication and encryption
    AES-256-GCM authenticated encryption for all stored artifacts
    Argon2id key derivation for master key from user password
    OS keychain storage for master key material
    OCR every ~5 seconds on changed frames (Tesseract)
    SQLite FTS5 full-text search over OCR index
    Timeline UI with thumbnail scrubbing
    Replay with seek across segment boundaries
    Event overlay during replay (mouse clicks, shortcuts, window changes)
    Clip export to standard container format (mp4/mkv)
    Cross-platform: Windows, macOS, Linux
    System tray integration with pause/lock controls
    Configurable skip/redact lists (apps, window title patterns, private browsing)

2.2 Non-Goals (Explicitly Out of Scope)

    ❌ Cloud upload, sync, or any remote storage
    ❌ AI/LLM inference beyond local OCR
    ❌ Network traffic inspection or browser proxy
    ❌ Remote viewing or web access to the archive
    ❌ Enterprise policy management or central admin
    ❌ Multi-user shared vaults
    ❌ Continuous raw audio recording
    ❌ Mobile devices (iOS/Android)

2.3 Constraints

    Capture pipeline must consume < 5% CPU on a modern machine at default settings
    Disk usage must be hard-capped by configuration (default 5 GB)
    Encryption overhead must not exceed 2ms per segment chunk
    OCR must never block the capture pipeline (async, separate task)
    UI must be responsive even if OCR/search indexing is behind
    All secret material must be wiped from memory on vault lock
    Segment file format must be stable and forward-compatible

3. System Architecture
3.1 High-Level Architecture Diagram

text

┌────────────────────────────────────────────────────────────────────────┐
│                          USER'S MACHINE                                │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     BLACKBOX LOGGER CORE (Rust/Tauri)           │  │
│  │                                                                  │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │  │
│  │  │   Screen    │  │   Input     │  │  Window/App │             │  │
│  │  │  Capture    │  │   Logger    │  │   Tracker   │             │  │
│  │  │  (OS APIs)  │  │   (rdev)    │  │  (OS APIs)  │             │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │  │
│  │         │                │                 │                    │  │
│  │         └────────────────┴─────────────────┘                   │  │
│  │                          │                                      │  │
│  │                ┌─────────▼─────────┐                           │  │
│  │                │   Event Bus       │                           │  │
│  │                │  (Tokio MPSC)     │                           │  │
│  │                └────────┬──────────┘                           │  │
│  │                         │                                      │  │
│  │         ┌───────────────┼───────────────┐                      │  │
│  │         │               │               │                      │  │
│  │  ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐              │  │
│  │  │   Video     │ │   Event     │ │  Thumbnail  │              │  │
│  │  │  Encoder    │ │   Store     │ │  Generator  │              │  │
│  │  │  (H.265)    │ │  (SQLite)   │ │             │              │  │
│  │  └──────┬──────┘ └─────────────┘ └──────┬──────┘              │  │
│  │         │                               │                      │  │
│  │  ┌──────▼──────────────────────────┐    │                      │  │
│  │  │       Segment Manager           │    │                      │  │
│  │  │   (Rolling ring buffer, prune)  │    │                      │  │
│  │  └──────┬──────────────────────────┘    │                      │  │
│  │         │                               │                      │  │
│  │  ┌──────▼──────┐               ┌────────▼──────┐               │  │
│  │  │  Crypto /   │               │  OCR Engine   │               │  │
│  │  │  Key Mgr    │               │  (Tesseract)  │               │  │
│  │  │  (AES-GCM)  │               └───────┬───────┘               │  │
│  │  └─────────────┘                       │                       │  │
│  │                               ┌────────▼──────┐                │  │
│  │                               │  SQLite FTS5  │                │  │
│  │                               │  Search Index │                │  │
│  │                               └───────────────┘                │  │
│  │                                                                  │  │
│  │  ┌──────────────────────────────────────────────────────────┐   │  │
│  │  │              Tauri IPC Bridge (Commands + Events)        │   │  │
│  │  └──────────────────────────────┬───────────────────────────┘   │  │
│  └──────────────────────────────────┼──────────────────────────────┘  │
│                                     │                                  │
│  ┌──────────────────────────────────▼──────────────────────────────┐  │
│  │              FRONTEND (Svelte + Tailwind + Tauri WebView)       │  │
│  │                                                                  │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │  │  Timeline   │  │   Search    │  │  Replay  │  │Settings  │  │  │
│  │  │    View     │  │  Results    │  │  Player  │  │  Panel   │  │  │
│  │  └─────────────┘  └─────────────┘  └──────────┘  └──────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

ON DISK:
  ~/.blackbox/segments/*.bbv          ← AES-256-GCM encrypted video chunks
  ~/.blackbox/thumbs/*.bbt            ← AES-256-GCM encrypted thumbnails
  ~/.blackbox/blackbox.db             ← SQLCipher-encrypted SQLite database

3.2 Data Flow (Step by Step)

text

Step 1:  Screen capture task produces VideoFrame at ~5 FPS
Step 2:  Frame sent via Tokio MPSC channel to encoder task
Step 3:  Encoder compresses frame with H.265 (FFmpeg bindings)
Step 4:  Segment manager groups compressed frames into 60s segments
Step 5:  Crypto module encrypts each segment chunk with AES-256-GCM
         (unique nonce per chunk, segment key encrypted by master key)
Step 6:  Encrypted chunk written to .bbv segment file on disk
Step 7:  Segment metadata (path, timestamps, keyframe index) written to DB
Step 8:  Thumbnail task samples every Nth frame → encrypts → writes .bbt
Step 9:  OCR scheduler checks if frame changed (vs last OCR'd frame)
         If changed: preprocess → Tesseract → postprocess → FTS5 insert
Step 10: Window tracker task writes focus changes to events table
Step 11: Input logger task writes redacted events to events table
Step 12: Cleaner task runs periodically, prunes segments beyond retention cap
Step 13: UI requests timeline → backend queries DB → returns thumbnail list
Step 14: User seeks to timestamp → backend decrypts segment range in-memory
         → streams frames to frontend → frontend renders in <canvas>
Step 15: User searches text → backend runs FTS5 query → returns results
         with thumbnail + timestamp + snippet → user clicks → replay seeks

3.3 Concurrency Model (Tokio Tasks)
Task	Channel	Priority	Notes
screen_capture_task	Sends VideoFrame to encoder	Real-time	Never block; drop frame if encoder full
encoder_task	Receives VideoFrame	High	H.265 encode + encrypt
window_tracker_task	Sends WindowEvent to event store	Medium	Polls every 250ms
input_logger_task	Sends InputEvent to event store	Medium	rdev hook thread
clipboard_task	Sends ClipboardEvent to event store	Low	Polls every 500ms
event_store_task	Receives all events, writes to DB	Medium	Serialized DB writes
thumbnail_task	Samples frames, generates thumbs	Low	Every Nth frame
ocr_task	Receives sampled frames, OCRs	Background	Async; skip if busy
cleaner_task	Prunes old segments	Very low	Runs every 60s
resource_monitor_task	Checks CPU/disk	Low	Throttles if needed

Backpressure rule: The capture task's MPSC channel must have a bounded buffer (e.g. 30 frames). If full, the capture task drops frames and emits a DroppedFrames(count) metric. Under no circumstance should frame dropping block or panic the capture task.
4. Module Breakdown
4.1 Screen Capture Engine

Purpose: Continuously capture screen frames at a configured FPS and resolution and emit VideoFrame structs into the event bus.

Default settings:

    FPS: 5
    Resolution: downscale to 1280×720
    Pixel format: RGBA before encode, YUV420P for H.265 input

Output type:

Rust

// capture/screen.rs

#[derive(Debug, Clone)]
pub struct VideoFrame {
    pub timestamp_utc: i64,         // Unix ms
    pub capture_stream_id: String,  // "monitor:0", "monitor:1", "mosaic"
    pub width: u32,
    pub height: u32,
    pub pixel_format: PixelFormat,
    pub data: Arc<Vec<u8>>,         // Arc: cheap clone for multi-consumer
    pub sequence_number: u64,
}

#[derive(Debug, Clone, Copy)]
pub enum PixelFormat {
    RGBA,
    BGRA,
    YUV420P,
}

Platform implementations:

Rust

// capture/screen.rs

#[cfg(target_os = "windows")]
mod platform {
    use windows::Win32::Graphics::Dxgi::*;
    // DXGI Desktop Duplication API
    // AcquireNextFrame → map to CPU → extract RGBA bytes
    pub struct DXGICaptureBackend { /* ... */ }
}

#[cfg(target_os = "macos")]
mod platform {
    // ScreenCaptureKit (preferred, macOS 12.3+)
    // SCShareableContent → SCStream → frame callbacks
    // Fallback: CGDisplayCreateImage (older macOS)
    pub struct ScreenCaptureKitBackend { /* ... */ }
}

#[cfg(target_os = "linux")]
mod platform {
    // Strategy: Try PipeWire/wlr-screencopy (Wayland) first
    // Fallback: x11rb XGetImage (X11)
    // Note: Wayland portal-based capture requires user mediation per-session
    pub struct LinuxCaptureBackend {
        backend_type: LinuxBackendType,
    }
    pub enum LinuxBackendType { PipeWire, X11 }
}

Multi-monitor strategy:

Rust

// capture/screen.rs

pub struct ScreenCaptureManager {
    monitors: Vec<MonitorCapture>,
    mode: CaptureMode,
}

pub enum CaptureMode {
    PrimaryOnly,
    AllMonitors,           // Independent streams per monitor
    Mosaic,                // Stitch all monitors side-by-side into one stream
    Specific(Vec<String>), // By monitor ID
}

impl ScreenCaptureManager {
    pub async fn run(&self, tx: mpsc::Sender<VideoFrame>) {
        match self.mode {
            CaptureMode::AllMonitors => {
                // Spawn one task per monitor, tag frames with capture_stream_id
                let handles: Vec<_> = self.monitors.iter()
                    .map(|m| tokio::spawn(m.clone().capture_loop(tx.clone())))
                    .collect();
                futures::future::join_all(handles).await;
            }
            // ...
        }
    }
}

FPS throttle logic:

Rust

// capture/screen.rs

pub async fn capture_loop(self, tx: mpsc::Sender<VideoFrame>) {
    let interval = Duration::from_millis(1000 / self.config.fps as u64);
    let mut ticker = tokio::time::interval(interval);
    let mut seq: u64 = 0;

    loop {
        ticker.tick().await;

        // If recording paused, skip
        if self.state.load(Ordering::Relaxed) == CaptureState::Paused {
            continue;
        }

        match self.backend.capture_frame() {
            Ok(frame) => {
                let vf = VideoFrame {
                    timestamp_utc: utc_now_ms(),
                    capture_stream_id: self.monitor_id.clone(),
                    width: frame.width,
                    height: frame.height,
                    pixel_format: frame.format,
                    data: Arc::new(frame.data),
                    sequence_number: seq,
                };
                seq += 1;

                // Non-blocking send: drop frame if encoder is busy
                if tx.try_send(vf).is_err() {
                    self.metrics.dropped_frames.fetch_add(1, Ordering::Relaxed);
                }
            }
            Err(e) => {
                tracing::error!(error = ?e, "frame capture error");
                tokio::time::sleep(Duration::from_millis(100)).await;
            }
        }
    }
}

4.2 Input Logger

Purpose: Capture keyboard and mouse events using platform-level hooks, apply redaction rules, and emit InputEvent structs for storage and replay overlay.

Critical design rule: Raw keystrokes are off by default. Only high-level events (shortcuts, clicks, scrolls) are captured unless the user explicitly enables keystroke capture in settings.

Rust

// capture/input.rs

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum InputEvent {
    MouseClick {
        timestamp: i64,
        button: MouseButton,
        x: i32,
        y: i32,
        window_id: Option<i64>,
    },
    MouseScroll {
        timestamp: i64,
        delta_x: f64,
        delta_y: f64,
    },
    MouseMove {
        timestamp: i64,
        x: i32,
        y: i32,
    },
    KeyPress {
        timestamp: i64,
        key: KeyDescription,
        is_redacted: bool,
        modifiers: KeyModifiers,
    },
    Shortcut {
        timestamp: i64,
        description: String, // e.g. "Ctrl+S", "Cmd+Z"
    },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum KeyDescription {
    Printable(char),
    Named(String),   // "Enter", "Backspace", "F5", etc.
    Redacted,        // Replaced when is_redacted = true
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KeyModifiers {
    pub ctrl: bool,
    pub alt: bool,
    pub shift: bool,
    pub meta: bool,  // Cmd on macOS, Win on Windows
}

Redaction logic:

Rust

// capture/input.rs

pub struct InputRedactor {
    config: RedactionConfig,
    current_window: Arc<RwLock<Option<WindowInfo>>>,
    password_mode: Arc<AtomicBool>,
}

impl InputRedactor {
    pub fn should_redact_keystrokes(&self) -> bool {
        // Always redact if raw keystroke capture is disabled
        if !self.config.capture_raw_keystrokes {
            return true;
        }

        // Redact if current app is in the redact-list
        if let Some(win) = self.current_window.read().unwrap().as_ref() {
            if self.config.redact_keystrokes_in_apps.iter()
                .any(|app| win.app_name.to_lowercase().contains(&app.to_lowercase()))
            {
                return true;
            }
        }

        // Redact if in detected password mode
        if self.password_mode.load(Ordering::Relaxed) {
            return true;
        }

        false
    }

    pub fn process_key_event(&self, raw: RawKeyEvent) -> Option<InputEvent> {
        // Mouse move throttle: only emit every 100ms
        if let RawKeyEvent::MouseMove { x, y, .. } = &raw {
            if self.last_mouse_move.elapsed() < Duration::from_millis(100) {
                return None;
            }
        }

        let is_redacted = self.should_redact_keystrokes();

        // Detect shortcuts regardless of redaction
        if let RawKeyEvent::KeyPress { key, mods, .. } = &raw {
            if mods.ctrl || mods.meta || mods.alt {
                return Some(InputEvent::Shortcut {
                    timestamp: utc_now_ms(),
                    description: format_shortcut(key, mods),
                });
            }
        }

        // Apply redaction to printable keys
        let key_desc = if is_redacted {
            KeyDescription::Redacted
        } else {
            raw.to_key_description()
        };

        Some(InputEvent::KeyPress {
            timestamp: utc_now_ms(),
            key: key_desc,
            is_redacted,
            modifiers: raw.modifiers(),
        })
    }
}

Platform note on Wayland: rdev input capture works reliably on X11 and Windows/macOS. On Wayland, global input hooks are not available without accessibility permissions or compositor-specific extensions. The system must detect this and:

    Notify the user in the permissions UI
    Fall back to "no input capture" gracefully without crashing

4.3 Window & App Tracker

Purpose: Maintain a continuous record of which application and window the user was focused on at every timestamp. This is the primary metadata for search filtering and replay context.

Rust

// capture/window.rs

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WindowInfo {
    pub window_id: i64,          // DB-assigned ID
    pub app_name: String,
    pub app_path: String,
    pub window_title: String,
    pub pid: u32,
    pub is_browser: bool,
    pub browser_url: Option<String>,  // Optional; requires browser extension
    pub first_seen: i64,
    pub last_seen: i64,
}

pub struct WindowTracker {
    current_window: Arc<RwLock<Option<WindowInfo>>>,
    db: Arc<Db>,
    config: WindowTrackerConfig,
    tx: mpsc::Sender<WindowEvent>,
}

impl WindowTracker {
    pub async fn run(&self) {
        let mut poll_interval = tokio::time::interval(
            Duration::from_millis(self.config.poll_interval_ms)
        );
        let mut last_window_id: Option<i64> = None;

        loop {
            poll_interval.tick().await;

            let current = match get_focused_window() {
                Ok(w) => w,
                Err(_) => continue,
            };

            // Skip capture if app is in the skip list
            if self.should_skip(&current) {
                continue;
            }

            let window_id = self.db.upsert_window(&current).await?;

            // Only emit event if window actually changed
            if Some(window_id) != last_window_id {
                self.tx.send(WindowEvent::Focus {
                    window_id,
                    timestamp: utc_now_ms(),
                }).await.ok();
                last_window_id = Some(window_id);
            } else {
                // Update last_seen timestamp
                self.db.update_window_last_seen(window_id, utc_now_ms()).await.ok();
            }
        }
    }

    fn should_skip(&self, window: &RawWindowInfo) -> bool {
        // Skip by app name
        if self.config.skip_apps.iter()
            .any(|app| window.app_name.to_lowercase().contains(&app.to_lowercase()))
        {
            return true;
        }

        // Skip by window title pattern (regex)
        for pattern in &self.config.skip_title_patterns {
            if let Ok(re) = regex::Regex::new(pattern) {
                if re.is_match(&window.title) {
                    return true;
                }
            }
        }

        // Skip private browsing (heuristic: title includes "Private" / "Incognito")
        if self.config.skip_private_browsing {
            let title_lower = window.title.to_lowercase();
            if title_lower.contains("private") || title_lower.contains("incognito")
                || title_lower.contains("inprivate")
            {
                return true;
            }
        }

        false
    }
}

Platform implementations:

Rust

#[cfg(target_os = "windows")]
fn get_focused_window() -> Result<RawWindowInfo> {
    use windows::Win32::UI::WindowsAndMessaging::*;
    use windows::Win32::System::Threading::*;
    let hwnd = unsafe { GetForegroundWindow() };
    // GetWindowTextW → title
    // GetWindowThreadProcessId → PID
    // QueryFullProcessImageNameW → app path
}

#[cfg(target_os = "macos")]
fn get_focused_window() -> Result<RawWindowInfo> {
    use core_foundation::*;
    // NSWorkspace.shared.frontmostApplication
    // AXUIElementCopyAttributeValue(kAXFocusedWindowAttribute) → title
}

#[cfg(target_os = "linux")]
fn get_focused_window() -> Result<RawWindowInfo> {
    // X11: XGetInputFocus + XFetchName + _NET_WM_PID
    // Wayland: limited; use XDG-portal or DBus if available
}

4.4 Clipboard Monitor

Purpose: Detect clipboard changes, deduplicate by hash, encrypt payload, and store for later context retrieval.

Rust

// capture/clipboard.rs

pub struct ClipboardMonitor {
    last_hash: Arc<Mutex<Option<[u8; 32]>>>,
    tx: mpsc::Sender<ClipboardEvent>,
    encryptor: Arc<Encryptor>,
    config: ClipboardConfig,
}

#[derive(Debug, Clone)]
pub struct ClipboardEvent {
    pub timestamp: i64,
    pub hash: [u8; 32],              // SHA-256 of plaintext (for dedup)
    pub encrypted_payload: Vec<u8>,  // AES-256-GCM encrypted text
    pub nonce: [u8; 12],
    pub tag: [u8; 16],
    pub content_type: ClipboardContentType,
    pub size_bytes: usize,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ClipboardContentType {
    PlainText,
    RichText,
    Image,  // Hash-only; image bytes not stored
    Other,
}

impl ClipboardMonitor {
    pub async fn run(&self) {
        let mut interval = tokio::time::interval(Duration::from_millis(500));
        loop {
            interval.tick().await;

            let content = match arboard::Clipboard::new().and_then(|mut c| c.get_text()) {
                Ok(text) => text,
                Err(_) => continue,
            };

            let hash = sha256(content.as_bytes());

            // Dedup: skip if same as last clipboard value
            let mut last = self.last_hash.lock().unwrap();
            if *last == Some(hash) {
                continue;
            }
            *last = Some(hash);

            // Encrypt before emitting
            let (ciphertext, nonce, tag) = self.encryptor
                .encrypt_bytes(content.as_bytes())
                .unwrap_or_else(|_| (vec![], [0u8; 12], [0u8; 16]));

            self.tx.send(ClipboardEvent {
                timestamp: utc_now_ms(),
                hash,
                encrypted_payload: ciphertext,
                nonce,
                tag,
                content_type: ClipboardContentType::PlainText,
                size_bytes: content.len(),
            }).await.ok();
        }
    }
}

4.5 Video Encoder & Segment Manager

Purpose: Accept raw VideoFrame structs from the capture bus, encode them to H.265, encrypt each chunk, and manage the rolling segment file structure on disk.
Encoder

Rust

// encoder/video_encoder.rs

use ffmpeg_next as ffmpeg;

pub struct H265Encoder {
    encoder: ffmpeg::codec::encoder::Video,
    scaler: ffmpeg::software::scaling::Context,
    config: EncoderConfig,
}

#[derive(Debug, Clone)]
pub struct EncoderConfig {
    pub width: u32,
    pub height: u32,
    pub fps: u32,
    pub crf: u32,          // Default 28; lower = higher quality + larger files
    pub preset: String,    // "ultrafast" → "veryslow"; default "veryfast"
    pub codec: VideoCodec,
}

#[derive(Debug, Clone)]
pub enum VideoCodec {
    H265,  // Default; best compression
    H264,  // Better compatibility if H265 issues arise
}

impl H265Encoder {
    pub fn new(config: EncoderConfig) -> Result<Self> {
        ffmpeg::init()?;
        let codec = ffmpeg::codec::encoder::find(ffmpeg::codec::Id::HEVC)
            .ok_or(EncoderError::CodecNotFound)?;
        let mut encoder = codec.video()?;

        encoder.set_width(config.width);
        encoder.set_height(config.height);
        encoder.set_frame_rate(Some((config.fps as i32, 1)));
        encoder.set_format(ffmpeg::format::pixel::Pixel::YUV420P);
        encoder.set_bit_rate(0);  // CRF mode; ignore bitrate

        // CRF and preset via private options
        let mut opts = ffmpeg::Dictionary::new();
        opts.set("crf", &config.crf.to_string());
        opts.set("preset", &config.preset);

        let encoder = encoder.open_with(opts)?;
        let scaler = ffmpeg::software::scaling::Context::get(
            ffmpeg::format::pixel::Pixel::RGBA,
            config.width, config.height,
            ffmpeg::format::pixel::Pixel::YUV420P,
            config.width, config.height,
            ffmpeg::software::scaling::Flags::BILINEAR,
        )?;

        Ok(H265Encoder { encoder, scaler, config })
    }

    pub fn encode_frame(&mut self, frame: &VideoFrame) -> Result<Vec<EncodedPacket>> {
        // Convert RGBA → YUV420P
        let yuv = self.scaler.run(&frame.to_ffmpeg_frame()?)?;

        // Encode
        self.encoder.send_frame(&yuv)?;

        let mut packets = Vec::new();
        let mut packet = ffmpeg::Packet::empty();
        while self.encoder.receive_packet(&mut packet).is_ok() {
            packets.push(EncodedPacket {
                data: packet.data().unwrap_or(&[]).to_vec(),
                pts: packet.pts().unwrap_or(0),
                dts: packet.dts().unwrap_or(0),
                is_keyframe: packet.is_key(),
                duration: packet.duration(),
            });
        }
        Ok(packets)
    }
}

Segment Manager

Rust

// encoder/segment_manager.rs

pub struct SegmentManager {
    config: SegmentConfig,
    current_segment: Option<ActiveSegment>,
    db: Arc<Db>,
    crypto: Arc<KeyManager>,
    metrics: Arc<Metrics>,
}

#[derive(Debug, Clone)]
pub struct SegmentConfig {
    pub segment_duration_secs: u64,   // Default 60
    pub storage_dir: PathBuf,
    pub max_disk_gb: f64,
    pub max_duration_hours: f64,
}

pub struct ActiveSegment {
    pub segment_id: Uuid,
    pub started_at: i64,
    pub capture_stream_id: String,
    pub file: SegmentFileWriter,
    pub keyframes: Vec<KeyframeEntry>,
    pub chunk_count: u32,
    pub segment_key: [u8; 32],       // Per-segment AES key
    pub nonce_base: [u8; 12],        // Base nonce for this segment
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KeyframeEntry {
    pub pts_ms: i64,
    pub byte_offset: u64,
    pub chunk_index: u32,
}

impl SegmentManager {
    pub async fn process_packet(&mut self, packet: EncodedPacket, stream_id: &str) -> Result<()> {
        // Start new segment if needed
        if self.should_rollover(&packet) {
            self.finalize_current_segment().await?;
            self.start_new_segment(stream_id).await?;
        }

        let seg = self.current_segment.as_mut()
            .ok_or(SegmentError::NoActiveSegment)?;

        // Record keyframe position
        if packet.is_keyframe {
            seg.keyframes.push(KeyframeEntry {
                pts_ms: packet.pts,
                byte_offset: seg.file.position(),
                chunk_index: seg.chunk_count,
            });
        }

        // Encrypt and write chunk
        let nonce = derive_chunk_nonce(&seg.nonce_base, seg.chunk_count as u64);
        let (ciphertext, tag) = aes_gcm_encrypt(&seg.segment_key, &nonce, &packet.data)?;

        seg.file.write_chunk(Chunk {
            counter: seg.chunk_count as u64,
            ciphertext,
            tag,
            nonce,
        })?;

        seg.chunk_count += 1;
        Ok(())
    }

    async fn finalize_current_segment(&mut self) -> Result<()> {
        if let Some(seg) = self.current_segment.take() {
            // Write footer with keyframe index
            seg.file.write_footer(&seg.keyframes)?;
            seg.file.flush()?;

            // Encrypt segment key with master key and store in DB
            let wrapped_key = self.crypto.wrap_segment_key(&seg.segment_key).await?;

            self.db.insert_segment(SegmentRecord {
                id: seg.segment_id,
                capture_stream_id: seg.capture_stream_id,
                started_at: seg.started_at,
                ended_at: utc_now_ms(),
                path: seg.file.path().to_string_lossy().into(),
                codec: "h265".into(),
                nonce_base: seg.nonce_base,
                wrapped_key,
                keyframe_index: serde_json::to_value(&seg.keyframes)?,
                chunk_count: seg.chunk_count,
                is_complete: true,
            }).await?;

            // Enforce retention limits
            self.enforce_retention().await?;
        }
        Ok(())
    }

    async fn enforce_retention(&self) -> Result<()> {
        // Prune by age
        let cutoff = utc_now_ms() - (self.config.max_duration_hours * 3600.0 * 1000.0) as i64;
        let old_segments = self.db.get_segments_before(cutoff).await?;

        // Prune by disk size
        let total_size = self.total_disk_usage().await?;
        let over_by = (total_size as f64 / 1e9) - self.config.max_disk_gb;

        for seg in old_segments {
            if over_by > 0.0 || seg.ended_at < cutoff {
                // Crypto-erase: delete key material first, then file
                self.db.delete_segment_key(&seg.id).await?;
                std::fs::remove_file(&seg.path).ok();
                self.db.delete_segment(&seg.id).await?;
            }
        }
        Ok(())
    }
}

Keyframe Index & Seek

Rust

// encoder/keyframe.rs

pub struct KeyframeIndex {
    entries: Vec<KeyframeEntry>,
}

impl KeyframeIndex {
    /// Find the nearest keyframe at or before target_pts_ms
    pub fn find_seek_point(&self, target_pts_ms: i64) -> Option<&KeyframeEntry> {
        self.entries
            .iter()
            .rev()
            .find(|kf| kf.pts_ms <= target_pts_ms)
    }
}

Thumbnail Generator

Rust

// encoder/thumbnail.rs

pub struct ThumbnailGenerator {
    config: ThumbnailConfig,
    encryptor: Arc<Encryptor>,
    db: Arc<Db>,
    frame_counter: u64,
}

pub struct ThumbnailConfig {
    pub width: u32,           // Default 320
    pub height: u32,          // Default 180
    pub every_n_frames: u32,  // At 5fps, every 10 frames = 2s density
    pub thumbs_dir: PathBuf,
}

impl ThumbnailGenerator {
    pub async fn maybe_generate(&mut self, frame: &VideoFrame, segment_id: Uuid) -> Result<()> {
        self.frame_counter += 1;
        if self.frame_counter % self.config.every_n_frames as u64 != 0 {
            return Ok(());
        }

        // Resize frame to thumbnail dimensions
        let thumb_bytes = resize_rgba_frame(
            &frame.data,
            frame.width, frame.height,
            self.config.width, self.config.height,
        )?;

        // Encode as JPEG
        let jpeg_bytes = encode_jpeg(&thumb_bytes, 75)?;

        // Encrypt and write
        let thumb_id = Uuid::new_v4();
        let path = self.config.thumbs_dir.join(format!("{}.bbt", thumb_id));
        let (ciphertext, nonce, tag) = self.encryptor.encrypt_bytes(&jpeg_bytes)?;
        write_encrypted_file(&path, &ciphertext, &nonce, &tag)?;

        // Store metadata
        self.db.insert_thumbnail(ThumbnailRecord {
            id: thumb_id,
            segment_id,
            capture_stream_id: frame.capture_stream_id.clone(),
            timestamp: frame.timestamp_utc,
            pts_ms: frame.timestamp_utc,
            window_id: None,
            path: path.to_string_lossy().into(),
            nonce,
            tag,
        }).await?;

        Ok(())
    }
}

4.6 Cryptography & Key Manager

Purpose: Derive and manage all encryption keys, wrap/unwrap segment keys, provide authenticated encryption and decryption primitives, and interface with the OS keychain.

See §9 for the full cryptography deep dive.

Rust

// crypto/key_manager.rs

pub struct KeyManager {
    master_key: Arc<RwLock<Option<[u8; 32]>>>,
    keychain: Box<dyn KeychainBackend>,
    db: Arc<Db>,
    config: EncryptionConfig,
}

pub trait KeychainBackend: Send + Sync {
    fn store_master_key(&self, key: &[u8]) -> Result<()>;
    fn load_master_key(&self) -> Result<Vec<u8>>;
    fn delete_master_key(&self) -> Result<()>;
}

impl KeyManager {
    /// Unlock vault: derive master key from password and cache in memory
    pub async fn unlock(&self, password: &str) -> Result<()> {
        let params = self.db.get_kdf_params().await?;
        let derived = argon2id_derive(
            password.as_bytes(),
            &params.salt,
            params.memory_kb,
            params.iterations,
            params.parallelism,
        )?;

        // Validate by decrypting vault header
        let header = self.db.get_vault_header().await?;
        let decrypted = aes_gcm_decrypt(&derived, &header.nonce, &header.ciphertext, &header.tag)?;
        if decrypted != VAULT_MAGIC {
            return Err(KeyError::WrongPassword);
        }

        *self.master_key.write().unwrap() = Some(derived);
        Ok(())
    }

    /// Lock vault: wipe master key and all cached segment keys from memory
    pub fn lock(&self) {
        let mut key = self.master_key.write().unwrap();
        if let Some(ref mut k) = *key {
            // Explicitly zero memory
            k.iter_mut().for_each(|b| *b = 0);
        }
        *key = None;
    }

    /// Generate a new per-segment key, wrap it with master key, return both
    pub async fn generate_segment_key(&self) -> Result<([u8; 32], Vec<u8>)> {
        let master = self.master_key.read().unwrap()
            .ok_or(KeyError::VaultLocked)?;

        let segment_key: [u8; 32] = rand::random();
        let nonce: [u8; 12] = rand::random();
        let (wrapped, _, tag) = aes_gcm_encrypt(&master, &nonce, &segment_key)?;

        // Store: nonce + ciphertext + tag
        let mut stored = Vec::with_capacity(12 + wrapped.len() + 16);
        stored.extend_from_slice(&nonce);
        stored.extend_from_slice(&wrapped);
        stored.extend_from_slice(&tag);

        Ok((segment_key, stored))
    }

    /// Unwrap a stored segment key using the master key
    pub async fn unwrap_segment_key(&self, wrapped: &[u8]) -> Result<[u8; 32]> {
        let master = self.master_key.read().unwrap()
            .ok_or(KeyError::VaultLocked)?;

        let nonce = &wrapped[..12];
        let tag = &wrapped[wrapped.len()-16..];
        let ciphertext = &wrapped[12..wrapped.len()-16];

        let plaintext = aes_gcm_decrypt(&master, nonce, ciphertext, tag)?;
        let mut key = [0u8; 32];
        key.copy_from_slice(&plaintext);
        Ok(key)
    }
}

4.7 OCR Engine & FTS Indexer

Purpose: Run Tesseract OCR on sampled video frames, postprocess results, and insert into SQLite FTS5 for full-text search.

Rust

// ocr/engine.rs

pub struct OcrEngine {
    tesseract: TesseractInstance,
    config: OcrConfig,
    last_frame_hash: Option<[u8; 32]>,
}

pub struct OcrConfig {
    pub enabled: bool,
    pub interval_secs: f64,    // Default 5.0
    pub languages: Vec<String>, // ["eng", "fra", ...]
    pub min_confidence: f32,    // Default 0.60
    pub skip_unchanged_frames: bool,
}

impl OcrEngine {
    pub async fn process_frame(&mut self, frame: &VideoFrame, ctx: OcrContext) -> Result<Option<OcrResult>> {
        if !self.config.enabled {
            return Ok(None);
        }

        // Change detection: skip if frame visually unchanged
        if self.config.skip_unchanged_frames {
            let hash = perceptual_hash(&frame.data, frame.width, frame.height);
            if Some(hash) == self.last_frame_hash {
                return Ok(None);
            }
            self.last_frame_hash = Some(hash);
        }

        // Preprocess frame for better OCR accuracy
        let preprocessed = self.preprocess(frame)?;

        // Run Tesseract
        let raw_result = self.tesseract.recognize(&preprocessed)?;

        // Postprocess: filter low confidence, normalize whitespace
        let filtered = self.postprocess(raw_result)?;

        if filtered.text.is_empty() {
            return Ok(None);
        }

        Ok(Some(OcrResult {
            timestamp: frame.timestamp_utc,
            segment_id: ctx.segment_id,
            capture_stream_id: frame.capture_stream_id.clone(),
            window_id: ctx.window_id,
            pts_ms: frame.timestamp_utc,
            text: filtered.text,
            confidence: filtered.confidence,
            word_count: filtered.word_count,
        }))
    }

    fn preprocess(&self, frame: &VideoFrame) -> Result<GrayImage> {
        // 1. Convert to grayscale
        let gray = to_grayscale(&frame.data, frame.width, frame.height);
        // 2. Denoise (Gaussian blur, mild)
        let denoised = gaussian_blur(&gray, 0.5);
        // 3. Optional upscale if resolution is very low
        let scaled = if frame.width < 1280 {
            upscale_2x(&denoised)
        } else {
            denoised
        };
        Ok(scaled)
    }

    fn postprocess(&self, raw: TesseractOutput) -> Result<FilteredOcrResult> {
        let words: Vec<&str> = raw.text.split_whitespace().collect();
        // Drop very short fragments (< 3 words)
        if words.len() < 3 {
            return Ok(FilteredOcrResult::empty());
        }
        // Drop low-confidence results
        if raw.mean_confidence < self.config.min_confidence {
            return Ok(FilteredOcrResult::empty());
        }
        // Normalize whitespace
        let normalized = words.join(" ");
        Ok(FilteredOcrResult {
            text: normalized,
            confidence: raw.mean_confidence,
            word_count: words.len(),
        })
    }
}

FTS5 Indexer:

Rust

// ocr/indexer.rs

pub struct FtsIndexer {
    db: Arc<Db>,
}

impl FtsIndexer {
    pub async fn index(&self, result: OcrResult) -> Result<()> {
        // Insert into content table
        let frame_id = self.db.insert_ocr_frame(OcrFrameRecord {
            id: Uuid::new_v4(),
            timestamp: result.timestamp,
            segment_id: result.segment_id,
            capture_stream_id: result.capture_stream_id.clone(),
            window_id: result.window_id,
            pts_ms: result.pts_ms,
            text: result.text.clone(),
            confidence: result.confidence,
        }).await?;

        // FTS5 virtual table is kept in sync via INSERT trigger
        // The trigger on ocr_frames → ocr_fts handles this automatically
        // (see §5.2 for trigger DDL)

        Ok(())
    }
}

4.8 Search Engine

Purpose: Accept structured search queries (text + app filter + time range) and return scored results with thumbnail references and text snippets.

Rust

// search/engine.rs

#[derive(Debug, Clone, Deserialize)]
pub struct SearchQuery {
    pub text: Option<String>,
    pub apps: Option<Vec<String>>,
    pub window_titles: Option<Vec<String>>,
    pub start_time: Option<i64>,   // Unix ms
    pub end_time: Option<i64>,     // Unix ms
    pub capture_stream_id: Option<String>,
    pub limit: Option<u32>,
    pub offset: Option<u32>,
}

#[derive(Debug, Clone, Serialize)]
pub struct SearchResult {
    pub rank: u32,
    pub score: f64,
    pub timestamp: i64,
    pub segment_id: Uuid,
    pub pts_ms: i64,
    pub capture_stream_id: String,
    pub thumbnail_path: Option<String>,
    pub thumbnail_nonce: Option<[u8; 12]>,
    pub thumbnail_tag: Option<[u8; 16]>,
    pub matched_text: String,        // Snippet with highlighted terms
    pub context_text: String,        // Surrounding context
    pub app_name: String,
    pub window_title: String,
    pub window_id: Option<i64>,
}

pub struct SearchEngine {
    db: Arc<Db>,
}

impl SearchEngine {
    pub async fn search(&self, query: SearchQuery) -> Result<Vec<SearchResult>> {
        let limit = query.limit.unwrap_or(50).min(200) as i64;
        let offset = query.offset.unwrap_or(0) as i64;

        // Build the SQL dynamically depending on which filters are provided
        if let Some(ref text) = query.text {
            self.text_search(text, &query, limit, offset).await
        } else {
            self.metadata_search(&query, limit, offset).await
        }
    }

    async fn text_search(
        &self,
        text: &str,
        query: &SearchQuery,
        limit: i64,
        offset: i64,
    ) -> Result<Vec<SearchResult>> {
        // Use FTS5 with bm25 ranking + highlight + snippet
        // JOIN to windows table for app/title filter
        // JOIN to thumbnails for nearest thumbnail

        let sql = r#"
            SELECT
                ocr_fts.rowid,
                bm25(ocr_fts) AS score,
                of.timestamp,
                of.segment_id,
                of.pts_ms,
                of.capture_stream_id,
                highlight(ocr_fts, 0, '<mark>', '</mark>') AS matched_text,
                snippet(ocr_fts, 0, '<mark>', '</mark>', '...', 32) AS context_text,
                w.app_name,
                w.window_title,
                w.id as window_id,
                t.path as thumb_path,
                t.nonce as thumb_nonce,
                t.tag as thumb_tag
            FROM ocr_fts
            JOIN ocr_frames of ON ocr_fts.rowid = of.id
            LEFT JOIN windows w ON of.window_id = w.id
            LEFT JOIN LATERAL (
                SELECT path, nonce, tag FROM thumbnails
                WHERE segment_id = of.segment_id
                ORDER BY ABS(timestamp - of.timestamp) ASC
                LIMIT 1
            ) t ON TRUE
            WHERE ocr_fts MATCH ?1
            AND (?2 IS NULL OR of.timestamp >= ?2)
            AND (?3 IS NULL OR of.timestamp <= ?3)
            AND (?4 IS NULL OR LOWER(w.app_name) LIKE '%' || LOWER(?4) || '%')
            ORDER BY score ASC  -- bm25 returns negative scores; ASC = best first
            LIMIT ?5 OFFSET ?6
        "#;

        self.db.query_search_results(sql, SearchParams {
            text: text.to_string(),
            start_time: query.start_time,
            end_time: query.end_time,
            app_filter: query.apps.as_ref().and_then(|a| a.first().cloned()),
            limit,
            offset,
        }).await
    }
}

4.9 Replay & Export Engine

Purpose: Locate segments containing a requested timestamp, decrypt them in-memory, seek using the keyframe index, and stream frames to the frontend. Also support export to standard video format.

Rust

// replay/player.rs

pub struct ReplaySession {
    pub session_id: Uuid,
    pub start_timestamp: i64,
    pub current_timestamp: i64,
    pub capture_stream_id: String,
    pub state: ReplayState,
    pub frame_tx: mpsc::Sender<ReplayFrame>,
}

#[derive(Debug, Clone, Serialize)]
pub struct ReplayFrame {
    pub session_id: Uuid,
    pub timestamp: i64,
    pub pts_ms: i64,
    pub width: u32,
    pub height: u32,
    pub data: Vec<u8>,  // RGBA; sent to frontend canvas
    pub events: Vec<OverlayEvent>,
}

pub struct ReplayEngine {
    db: Arc<Db>,
    key_manager: Arc<KeyManager>,
}

impl ReplayEngine {
    pub async fn seek(&self, session: &mut ReplaySession, target_ts: i64) -> Result<()> {
        // 1. Find segment containing target_ts
        let segment = self.db.find_segment_at(target_ts, &session.capture_stream_id).await?
            .ok_or(ReplayError::NoSegmentAtTimestamp(target_ts))?;

        // 2. Unwrap segment key
        let segment_key = self.key_manager
            .unwrap_segment_key(&segment.wrapped_key).await?;

        // 3. Find nearest keyframe at or before target
        let keyframes: Vec<KeyframeEntry> = serde_json::from_value(segment.keyframe_index)?;
        let kf_index = KeyframeIndex { entries: keyframes };
        let seek_point = kf_index.find_seek_point(target_ts - segment.started_at)
            .ok_or(ReplayError::NoKeyframeFound)?;

        // 4. Read and decrypt from seek point onward
        let file = std::fs::File::open(&segment.path)?;
        let decrypted_stream = self.decrypt_from_offset(
            file,
            &segment_key,
            &segment.nonce_base,
            seek_point.chunk_index,
            seek_point.byte_offset,
        )?;

        // 5. Decode frames until we reach exact target PTS
        let decoder = H265Decoder::new(segment.width, segment.height)?;
        for packet in decrypted_stream {
            let frames = decoder.decode(&packet.data)?;
            for frame in frames {
                let abs_ts = segment.started_at + frame.pts_ms;
                if abs_ts >= target_ts {
                    // Query events for overlay
                    let events = self.db.get_events_in_range(abs_ts - 500, abs_ts + 500).await?;
                    session.frame_tx.send(ReplayFrame {
                        session_id: session.session_id,
                        timestamp: abs_ts,
                        pts_ms: frame.pts_ms,
                        width: frame.width,
                        height: frame.height,
                        data: frame.rgba_data,
                        events: events.into_iter().map(Into::into).collect(),
                    }).await.ok();
                }
            }
        }
        Ok(())
    }
}

Export engine:

Rust

// replay/exporter.rs

pub struct ExportConfig {
    pub start_ts: i64,
    pub end_ts: i64,
    pub output_path: PathBuf,
    pub format: ExportFormat,
    pub include_events_overlay: bool,
    pub capture_stream_id: String,
}

pub enum ExportFormat {
    Mp4,
    Mkv,
    WebM,
}

pub struct Exporter {
    db: Arc<Db>,
    key_manager: Arc<KeyManager>,
}

impl Exporter {
    pub async fn export(&self, config: ExportConfig, progress_tx: mpsc::Sender<f64>) -> Result<()> {
        // 1. Collect all segments overlapping [start_ts, end_ts]
        let segments = self.db.find_segments_in_range(
            config.start_ts, config.end_ts, &config.capture_stream_id
        ).await?;

        let total_segments = segments.len() as f64;

        // 2. Initialize output muxer
        let mut muxer = FFmpegMuxer::new(&config.output_path, config.format)?;

        for (i, segment) in segments.iter().enumerate() {
            // 3. Decrypt segment in memory
            let segment_key = self.key_manager.unwrap_segment_key(&segment.wrapped_key).await?;
            let packets = self.decrypt_segment_range(segment, &segment_key, config.start_ts, config.end_ts)?;

            // 4. If overlay requested: decode → composite → re-encode
            if config.include_events_overlay {
                let events = self.db.get_events_in_range(
                    config.start_ts, config.end_ts
                ).await?;
                for packet in packets {
                    let frames = decode_packets(&[packet])?;
                    for frame in frames {
                        let abs_ts = segment.started_at + frame.pts_ms;
                        let active_events = events.iter()
                            .filter(|e| (e.timestamp - abs_ts).abs() < 500)
                            .collect::<Vec<_>>();
                        let composited = composite_overlay(&frame, &active_events)?;
                        muxer.write_frame(&composited)?;
                    }
                }
            } else {
                // 5. Pass-through: just remux without re-encoding
                for packet in packets {
                    muxer.write_packet(&packet)?;
                }
            }

            progress_tx.send((i + 1) as f64 / total_segments).await.ok();
        }

        muxer.finalize()?;

        // 6. Record in saved_clips table
        self.db.insert_saved_clip(SavedClipRecord {
            id: Uuid::new_v4(),
            created_at: utc_now_ms(),
            start_ts: config.start_ts,
            end_ts: config.end_ts,
            output_path: config.output_path.to_string_lossy().into(),
            format: format!("{:?}", config.format),
            capture_stream_id: config.capture_stream_id,
        }).await?;

        Ok(())
    }
}

4.10 Tauri IPC Command Layer

Purpose: Expose all backend functionality to the Svelte frontend through Tauri's command system.

Rust

// commands/capture_commands.rs

#[tauri::command]
pub async fn capture_start(state: tauri::State<'_, AppState>) -> Result<(), String> {
    state.capture_manager.start().await.map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn capture_pause(state: tauri::State<'_, AppState>) -> Result<(), String> {
    state.capture_manager.pause().await.map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn capture_status(state: tauri::State<'_, AppState>) -> Result<CaptureStatus, String> {
    Ok(state.capture_manager.status())
}

#[tauri::command]
pub async fn vault_unlock(
    password: String,
    state: tauri::State<'_, AppState>,
) -> Result<(), String> {
    state.key_manager.unlock(&password).await.map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn vault_lock(state: tauri::State<'_, AppState>) -> Result<(), String> {
    state.key_manager.lock();
    state.capture_manager.pause().await.map_err(|e| e.to_string())
}

// commands/search_commands.rs

#[tauri::command]
pub async fn search(
    query: SearchQuery,
    state: tauri::State<'_, AppState>,
) -> Result<Vec<SearchResult>, String> {
    state.search_engine.search(query).await.map_err(|e| e.to_string())
}

// commands/replay_commands.rs

#[tauri::command]
pub async fn replay_create_session(
    timestamp: i64,
    capture_stream_id: Option<String>,
    state: tauri::State<'_, AppState>,
    app_handle: tauri::AppHandle,
) -> Result<String, String> {
    let session_id = state.replay_engine
        .create_session(timestamp, capture_stream_id)
        .await
        .map_err(|e| e.to_string())?;
    Ok(session_id.to_string())
}

#[tauri::command]
pub async fn replay_seek(
    session_id: String,
    timestamp: i64,
    state: tauri::State<'_, AppState>,
) -> Result<(), String> {
    let id = Uuid::parse_str(&session_id).map_err(|e| e.to_string())?;
    state.replay_engine.seek(id, timestamp).await.map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn export_clip(
    start_ts: i64,
    end_ts: i64,
    include_events: bool,
    output_path: Option<String>,
    state: tauri::State<'_, AppState>,
    app_handle: tauri::AppHandle,
) -> Result<String, String> {
    // ...
}

// commands/settings_commands.rs

#[tauri::command]
pub async fn settings_get(state: tauri::State<'_, AppState>) -> Result<AppSettings, String> {
    Ok(state.config.read().unwrap().clone().into())
}

#[tauri::command]
pub async fn settings_set(
    new_settings: PartialSettings,
    state: tauri::State<'_, AppState>,
) -> Result<AppSettings, String> {
    // ...
}

4.11 Dashboard UI (Svelte)

Purpose: Provide a full-featured graphical interface for timeline review, search, replay, settings, and export.
Page 1: Timeline View

text

┌──────────────────────────────────────────────────────────────────────┐
│  🎬 BLACKBOX LOGGER  │ ● RECORDING  │ Buffer: 1.2 GB / 5 GB          │
├──────────────────────────────────────────────────────────────────────┤
│  [🔍 Search everything...]    [Today ▼]  [All Apps ▼]  [⚙ Settings]  │
│  [⏸ Pause]  [🔒 Lock Vault]  [📤 Export Clip]                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ── TODAY ────────────────────────────────────────────────────────   │
│                                                                      │
│  14:32 ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐   │
│        │  VS Code  │  │  Chrome   │  │ Terminal  │  │  Figma   │   │
│        │ [thumb]   │  │ [thumb]   │  │ [thumb]   │  │ [thumb]  │   │
│        │ 14:32-    │  │ 14:40-    │  │ 14:52-    │  │ 15:01-   │   │
│        │ 14:40     │  │ 14:52     │  │ 15:01     │  │ 15:18    │   │
│        └───────────┘  └───────────┘  └───────────┘  └──────────┘   │
│                                                                      │
│  ── YESTERDAY ─────────────────────────────────────────────────────  │
│                                                                      │
│  [Load more...]                                                      │
├──────────────────────────────────────────────────────────────────────┤
│  Oldest: 48h ago  │  OCR Index: ✅ Current  │  Retention: 48h / 5GB  │
└──────────────────────────────────────────────────────────────────────┘

Page 2: Search Results View

text

┌──────────────────────────────────────────────────────────────────────┐
│  🔍 "forgot to save"   [Clear]              3 results in 0.04s       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  [thumb]   VS Code  •  Yesterday 14:37                         │  │
│  │            "...document.txt — you <mark>forgot to save</mark>  │  │
│  │            the changes before closing..."                       │  │
│  │            [▶ Jump Here]  [📋 Copy Text]  [✂ Clip]            │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  [thumb]   Chrome  •  Yesterday 11:22                          │  │
│  │            "Did you <mark>forget to save</mark> your work?"    │  │
│  │            [▶ Jump Here]  [📋 Copy Text]  [✂ Clip]            │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

Page 3: Replay Player View

text

┌──────────────────────────────────────────────────────────────────────┐
│  ▶ REPLAY  │  VS Code  •  Yesterday 14:37:22  │  [✂ Clip from here]  │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                                                                │  │
│  │                   [VIDEO CANVAS 16:9]                          │  │
│  │                   Replay frame here                            │  │
│  │                  🖱 click overlay  ⌨ Ctrl+S                    │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ├──────────────────●──────────────────────────────────────────────┤  │
│  14:30             14:37                                     15:00  │
│                                                                      │
│  [◀◀ -30s]  [◀ -5s]  [⏸ Pause]  [▶ +5s]  [▶▶ +30s]  [1.0x ▼]     │
├──────────────────────────────────────────────────────────────────────┤
│  📋 OCR at this moment:  "main.rs — unsaved changes — line 142"      │
│  🪟 Window: VS Code — main.rs  │  ⌨ Last input: Ctrl+Z  (14:37:19)  │
└──────────────────────────────────────────────────────────────────────┘

Page 4: Settings Panel

text

┌──────────────────────────────────────────────────────────────────────┐
│  ⚙ SETTINGS                                                          │
├──────────────────────────────────────────────────────────────────────┤
│  CAPTURE                                                             │
│  FPS:          [5 ▼]    Resolution: [1280×720 ▼]    Quality: [28]    │
│  Monitors:     [Primary Only ▼]     Audio: [Off ▼]                   │
│                                                                      │
│  BUFFER                                                              │
│  Max Duration: [48 hours ▼]         Max Disk:  [5 GB]                │
│  Storage Path: [~/.blackbox/   ] [Browse]                            │
│                                                                      │
│  OCR & SEARCH                                                        │
│  OCR Enabled:  [✅]     Interval: [5s]     Languages: [eng]          │
│  Min Confidence: [0.60]                                              │
│                                                                      │
│  PRIVACY                                                             │
│  Skip Apps:    [1Password, Bitwarden ...]        [+ Add App]         │
│  Skip Patterns:[Private, Incognito, InPrivate]   [+ Add Pattern]     │
│  Redact Keys In: [1Password, Terminal ...]       [+ Add App]         │
│  Skip Private Browsing: [✅]                                          │
│                                                                      │
│  VAULT & ENCRYPTION                                                  │
│  Algorithm:    [AES-256-GCM]   KDF: [Argon2id]                       │
│  Auto-Lock:    [60 minutes]                                          │
│  [Change Password]  [Export Public Key]  [Delete All Data]           │
│                                                                      │
│  PERFORMANCE                                                         │
│  Max CPU:      [10%]    Pause on Battery: [✅]    Priority: [Low]     │
│                                                                      │
│  [Save Settings]                                                     │
└──────────────────────────────────────────────────────────────────────┘

Svelte stores:

TypeScript

// src/lib/stores/timeline.ts
import { writable, derived } from 'svelte/store';
import type { TimelineGroup, Thumbnail } from '../types';

export const timelineGroups = writable<TimelineGroup[]>([]);
export const selectedTimestamp = writable<number | null>(null);
export const activeFilters = writable({ app: '', dateRange: 'today' });

// src/lib/stores/search.ts
export const searchQuery = writable('');
export const searchResults = writable<SearchResult[]>([]);
export const searchLoading = writable(false);

// src/lib/stores/player.ts
export const activeSession = writable<string | null>(null);
export const currentFrame = writable<ReplayFrame | null>(null);
export const playerState = writable<'idle' | 'playing' | 'seeking' | 'paused'>('idle');

// src/lib/stores/capture.ts
export const captureStatus = writable<CaptureStatus | null>(null);
export const vaultLocked = writable(true);

Tauri event listener for real-time push:

TypeScript

// src/lib/api/events.ts
import { listen } from '@tauri-apps/api/event';

export function setupEventListeners() {
  listen<CaptureStatus>('bb/status', ({ payload }) => {
    captureStatus.set(payload);
  });

  listen<OcrProgressEvent>('bb/ocr_progress', ({ payload }) => {
    // Update OCR index status in UI
  });

  listen<ExportProgressEvent>('bb/export_progress', ({ payload }) => {
    exportProgress.set(payload.percent);
  });

  listen<ReplayFrame>('bb/replay_frame', ({ payload }) => {
    currentFrame.set(payload);
    renderFrameToCanvas(payload);
  });

  listen<ErrorEvent>('bb/error', ({ payload }) => {
    addToast({ type: 'error', message: payload.message });
  });
}

4.12 System Integration (Tray, Autostart, Permissions)

Rust

// system/tray.rs

pub fn build_tray(app: &tauri::App) -> tauri::SystemTray {
    let pause_resume = CustomMenuItem::new("pause_resume", "⏸ Pause Recording");
    let lock = CustomMenuItem::new("lock", "🔒 Lock Vault");
    let open = CustomMenuItem::new("open", "📁 Open Timeline");
    let search = CustomMenuItem::new("search", "🔍 Search");
    let export = CustomMenuItem::new("export", "📤 Export Last 5 min");
    let quit = CustomMenuItem::new("quit", "Quit");

    let menu = SystemTrayMenu::new()
        .add_item(pause_resume)
        .add_item(lock)
        .add_native_item(SystemTrayMenuItem::Separator)
        .add_item(open)
        .add_item(search)
        .add_item(export)
        .add_native_item(SystemTrayMenuItem::Separator)
        .add_item(quit);

    SystemTray::new().with_menu(menu)
}

// system/permissions.rs

pub struct PermissionsChecker;

impl PermissionsChecker {
    pub fn check_all() -> PermissionsReport {
        PermissionsReport {
            screen_recording: Self::check_screen_recording(),
            accessibility: Self::check_accessibility(),
            notifications: Self::check_notifications(),
        }
    }

    #[cfg(target_os = "macos")]
    fn check_screen_recording() -> PermissionStatus {
        // CGPreflightScreenCaptureAccess()
        // Returns .granted / .denied / .notDetermined
    }

    #[cfg(target_os = "windows")]
    fn check_screen_recording() -> PermissionStatus {
        // DXGI Desktop Duplication doesn't require explicit permission
        // but may need elevation for certain targets
        PermissionStatus::Granted
    }
}

5. Data Models & Schemas
5.1 SQLite Schema (Full DDL)

SQL

-- migrations/001_initial.sql

-- Vault metadata (tracks KDF params and vault header for unlock validation)
CREATE TABLE IF NOT EXISTS vault_metadata (
    id              INTEGER PRIMARY KEY CHECK (id = 1),  -- Singleton row
    kdf_salt        BLOB NOT NULL,
    argon2_memory_kb INTEGER NOT NULL DEFAULT 65536,
    argon2_iterations INTEGER NOT NULL DEFAULT 3,
    argon2_parallelism INTEGER NOT NULL DEFAULT 1,
    kdf_version     INTEGER NOT NULL DEFAULT 1,
    vault_header_ciphertext BLOB NOT NULL,
    vault_header_nonce      BLOB NOT NULL,
    vault_header_tag        BLOB NOT NULL,
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL
);

-- Segment files on disk
CREATE TABLE IF NOT EXISTS segments (
    id              TEXT PRIMARY KEY,        -- UUID
    capture_stream_id TEXT NOT NULL,         -- "monitor:0", "monitor:1", "mosaic"
    started_at      INTEGER NOT NULL,        -- Unix ms
    ended_at        INTEGER,
    duration_ms     INTEGER,
    path            TEXT NOT NULL,
    codec           TEXT NOT NULL DEFAULT 'h265',
    width           INTEGER NOT NULL,
    height          INTEGER NOT NULL,
    fps             REAL NOT NULL,
    nonce_base      BLOB NOT NULL,           -- 12 bytes: base nonce for chunk derivation
    wrapped_key     BLOB NOT NULL,           -- AES-GCM-wrapped segment key
    chunk_count     INTEGER NOT NULL DEFAULT 0,
    keyframe_index  TEXT,                    -- JSON: [{pts_ms, byte_offset, chunk_index}]
    is_complete     INTEGER NOT NULL DEFAULT 0,
    file_size_bytes INTEGER,
    created_at      INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- All captured events (keyboard, mouse, window focus, clipboard)
CREATE TABLE IF NOT EXISTS events (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp       INTEGER NOT NULL,
    segment_id      TEXT REFERENCES segments(id) ON DELETE CASCADE,
    capture_stream_id TEXT,
    event_type      TEXT NOT NULL,  -- 'key_press','mouse_click','mouse_scroll',
                                    --  'window_focus','clipboard_change','shortcut'
    -- Key events
    key_char        TEXT,           -- NULL if is_redacted=1
    key_name        TEXT,           -- "Enter", "Backspace", etc.
    key_mods        TEXT,           -- JSON: {"ctrl":false,"alt":false,...}
    is_redacted     INTEGER NOT NULL DEFAULT 0,
    -- Mouse events
    mouse_x         INTEGER,
    mouse_y         INTEGER,
    mouse_button    TEXT,
    mouse_delta_x   REAL,
    mouse_delta_y   REAL,
    -- Shortcut (high-level summary)
    shortcut_desc   TEXT,
    -- Window context
    window_id       INTEGER REFERENCES windows(id),
    -- Clipboard
    clipboard_hash  BLOB,           -- SHA-256, for dedup
    clipboard_nonce BLOB,
    clipboard_tag   BLOB,
    clipboard_ciphertext BLOB
);

-- Window/app focus history
CREATE TABLE IF NOT EXISTS windows (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    app_name        TEXT NOT NULL,
    app_path        TEXT,
    window_title    TEXT,
    pid             INTEGER,
    is_browser      INTEGER NOT NULL DEFAULT 0,
    browser_url     TEXT,           -- Optional
    first_seen      INTEGER NOT NULL,
    last_seen       INTEGER NOT NULL
);

-- Thumbnail files on disk
CREATE TABLE IF NOT EXISTS thumbnails (
    id              TEXT PRIMARY KEY,        -- UUID
    segment_id      TEXT NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
    capture_stream_id TEXT NOT NULL,
    timestamp       INTEGER NOT NULL,
    pts_ms          INTEGER NOT NULL,
    window_id       INTEGER REFERENCES windows(id),
    path            TEXT NOT NULL,           -- .bbt encrypted thumbnail
    nonce           BLOB NOT NULL,
    tag             BLOB NOT NULL,
    width           INTEGER NOT NULL DEFAULT 320,
    height          INTEGER NOT NULL DEFAULT 180
);

-- OCR results (content table for FTS5)
CREATE TABLE IF NOT EXISTS ocr_frames (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp       INTEGER NOT NULL,
    segment_id      TEXT REFERENCES segments(id) ON DELETE CASCADE,
    capture_stream_id TEXT,
    window_id       INTEGER REFERENCES windows(id),
    pts_ms          INTEGER NOT NULL,
    text            TEXT NOT NULL,
    confidence      REAL NOT NULL,
    word_count      INTEGER NOT NULL DEFAULT 0
);

-- FTS5 virtual table (full-text search over OCR output)
CREATE VIRTUAL TABLE IF NOT EXISTS ocr_fts USING fts5(
    text,
    content='ocr_frames',
    content_rowid='id',
    tokenize='unicode61 remove_diacritics 1'
);

-- Triggers to keep FTS5 in sync with content table
CREATE TRIGGER IF NOT EXISTS ocr_frames_ai
    AFTER INSERT ON ocr_frames BEGIN
        INSERT INTO ocr_fts(rowid, text) VALUES (new.id, new.text);
    END;

CREATE TRIGGER IF NOT EXISTS ocr_frames_ad
    AFTER DELETE ON ocr_frames BEGIN
        INSERT INTO ocr_fts(ocr_fts, rowid, text) VALUES ('delete', old.id, old.text);
    END;

CREATE TRIGGER IF NOT EXISTS ocr_frames_au
    AFTER UPDATE ON ocr_frames BEGIN
        INSERT INTO ocr_fts(ocr_fts, rowid, text) VALUES ('delete', old.id, old.text);
        INSERT INTO ocr_fts(rowid, text) VALUES (new.id, new.text);
    END;

-- User-defined privacy rules (skip/redact)
CREATE TABLE IF NOT EXISTS redaction_rules (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    rule_type       TEXT NOT NULL,  -- 'skip_app','skip_title_pattern','redact_keys_in_app'
    pattern         TEXT NOT NULL,
    is_regex        INTEGER NOT NULL DEFAULT 0,
    enabled         INTEGER NOT NULL DEFAULT 1,
    created_at      INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

-- Explicitly saved/exported clips
CREATE TABLE IF NOT EXISTS saved_clips (
    id              TEXT PRIMARY KEY,
    created_at      INTEGER NOT NULL,
    start_ts        INTEGER NOT NULL,
    end_ts          INTEGER NOT NULL,
    output_path     TEXT NOT NULL,
    format          TEXT NOT NULL,
    capture_stream_id TEXT NOT NULL,
    note            TEXT
);

-- App settings (key/value)
CREATE TABLE IF NOT EXISTS settings (
    key             TEXT PRIMARY KEY,
    value           TEXT NOT NULL,
    updated_at      INTEGER NOT NULL DEFAULT (unixepoch() * 1000)
);

5.2 Required Indexes

SQL

-- migrations/002_indexes.sql

-- Hot path: timeline queries
CREATE INDEX idx_segments_started_at        ON segments(started_at);
CREATE INDEX idx_segments_ended_at          ON segments(ended_at);
CREATE INDEX idx_segments_stream_time       ON segments(capture_stream_id, started_at, ended_at);

-- Event queries
CREATE INDEX idx_events_timestamp           ON events(timestamp);
CREATE INDEX idx_events_segment_id          ON events(segment_id);
CREATE INDEX idx_events_window_id           ON events(window_id);
CREATE INDEX idx_events_type_time           ON events(event_type, timestamp);

-- Window queries
CREATE INDEX idx_windows_app_name           ON windows(app_name);
CREATE INDEX idx_windows_last_seen          ON windows(last_seen);

-- Thumbnail queries
CREATE INDEX idx_thumbnails_timestamp       ON thumbnails(timestamp);
CREATE INDEX idx_thumbnails_segment_id      ON thumbnails(segment_id);
CREATE INDEX idx_thumbnails_stream_time     ON thumbnails(capture_stream_id, timestamp);

-- OCR queries
CREATE INDEX idx_ocr_frames_timestamp       ON ocr_frames(timestamp);
CREATE INDEX idx_ocr_frames_segment_id      ON ocr_frames(segment_id);
CREATE INDEX idx_ocr_frames_window_id       ON ocr_frames(window_id);
CREATE INDEX idx_ocr_frames_stream_time     ON ocr_frames(capture_stream_id, timestamp);

6. API Specifications (Tauri Commands & Events)
6.1 Backend → Frontend Events

All emitted via app_handle.emit_all():
Event Name	Payload Type	Description
bb/status	CaptureStatus	Periodic status broadcast (every 2s)
bb/replay_frame	ReplayFrame	Decoded frame bytes for canvas render
bb/ocr_progress	OcrProgressEvent	OCR backlog size + indexing ETA
bb/export_progress	ExportProgressEvent	0.0–1.0 progress fraction
bb/error	ErrorEvent	Non-fatal errors for user notification
bb/vault_locked	{}	Vault has been locked (auto-lock timer)
bb/permission_state_changed	PermissionsReport	Platform permissions changed
bb/segment_finalized	SegmentSummary	New segment written; UI can refresh
6.2 Frontend → Backend Commands (Full Spec)

TypeScript

// src/lib/api/client.ts

import { invoke } from '@tauri-apps/api/tauri';

// ── Capture ──────────────────────────────────────────────────────────

export const captureStart = () =>
    invoke<void>('capture_start');

export const capturePause = () =>
    invoke<void>('capture_pause');

export const captureResume = () =>
    invoke<void>('capture_resume');

export const captureStatus = () =>
    invoke<CaptureStatus>('capture_status');

// ── Vault ────────────────────────────────────────────────────────────

export const vaultUnlock = (password: string) =>
    invoke<void>('vault_unlock', { password });

export const vaultLock = () =>
    invoke<void>('vault_lock');

export const vaultChangePassword = (oldPassword: string, newPassword: string) =>
    invoke<void>('vault_change_password', { oldPassword, newPassword });

// ── Timeline ─────────────────────────────────────────────────────────

export const getTimeline = (params: TimelineParams) =>
    invoke<TimelineResponse>('get_timeline', { params });
    // Returns: { groups: [{ date, items: [{ timestamp, thumbnail_id, app_name, window_title }] }] }

export const getThumbnail = (thumbnailId: string) =>
    invoke<string>('get_thumbnail_base64', { thumbnailId });
    // Returns: base64-encoded JPEG (decrypted in backend, never touches disk unencrypted)

// ── Search ───────────────────────────────────────────────────────────

export const search = (query: SearchQuery) =>
    invoke<SearchResult[]>('search', { query });

// ── Replay ───────────────────────────────────────────────────────────

export const replayCreateSession = (timestamp: number, captureStreamId?: string) =>
    invoke<string>('replay_create_session', { timestamp, captureStreamId });

export const replaySeek = (sessionId: string, timestamp: number) =>
    invoke<void>('replay_seek', { sessionId, timestamp });

export const replayPlay = (sessionId: string) =>
    invoke<void>('replay_play', { sessionId });

export const replayPause = (sessionId: string) =>
    invoke<void>('replay_pause_session', { sessionId });

export const replayClose = (sessionId: string) =>
    invoke<void>('replay_close_session', { sessionId });

// ── Export ───────────────────────────────────────────────────────────

export const exportClip = (params: ExportParams) =>
    invoke<string>('export_clip', { ...params });
    // Returns: output file path

// ── Settings ─────────────────────────────────────────────────────────

export const settingsGet = () =>
    invoke<AppSettings>('settings_get');

export const settingsSet = (settings: Partial<AppSettings>) =>
    invoke<AppSettings>('settings_set', { newSettings: settings });

// ── System ───────────────────────────────────────────────────────────

export const checkPermissions = () =>
    invoke<PermissionsReport>('check_permissions');

export const requestPermission = (permission: PermissionType) =>
    invoke<void>('request_permission', { permission });

export const deleteAllData = () =>
    invoke<void>('delete_all_data');

6.3 TypeScript Types

TypeScript

// src/lib/types/index.ts

export interface CaptureStatus {
    recording: boolean;
    fps_actual: number;
    fps_target: number;
    dropped_frames: number;
    buffer_used_gb: number;
    buffer_max_gb: number;
    oldest_timestamp: number | null;  // Unix ms
    ocr_queue_size: number;
    vault_locked: boolean;
    active_monitors: string[];
}

export interface SearchQuery {
    text?: string;
    apps?: string[];
    window_titles?: string[];
    start_time?: number;   // Unix ms
    end_time?: number;
    capture_stream_id?: string;
    limit?: number;
    offset?: number;
}

export interface SearchResult {
    rank: number;
    score: number;
    timestamp: number;
    segment_id: string;
    pts_ms: number;
    capture_stream_id: string;
    thumbnail_id?: string;
    matched_text: string;
    context_text: string;
    app_name: string;
    window_title: string;
    window_id?: number;
}

export interface ReplayFrame {
    session_id: string;
    timestamp: number;
    pts_ms: number;
    width: number;
    height: number;
    data: number[];   // RGBA bytes; render to canvas
    events: OverlayEvent[];
}

export interface OverlayEvent {
    timestamp: number;
    event_type: string;
    description: string;
    x?: number;
    y?: number;
}

export interface AppSettings {
    capture: CaptureSettings;
    buffer: BufferSettings;
    ocr: OcrSettings;
    privacy: PrivacySettings;
    encryption: EncryptionSettings;
    ui: UiSettings;
    performance: PerformanceSettings;
}

7. Directory Structure

text

blackbox-logger/
│
├── src-tauri/                          ← Rust backend (Tauri)
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   ├── build.rs
│   └── src/
│       ├── main.rs                     ← Entry point; wires all modules
│       ├── lib.rs                      ← AppState struct; Tauri builder
│       │
│       ├── capture/
│       │   ├── mod.rs
│       │   ├── screen.rs               ← VideoFrame producer; platform impls
│       │   ├── input.rs                ← rdev hook; InputEvent + redaction
│       │   ├── window.rs               ← Window/app focus tracker
│       │   ├── clipboard.rs            ← Clipboard change monitor
│       │   ├── audio.rs                ← Optional audio (disabled by default)
│       │   └── manager.rs              ← CaptureManager; orchestrates all tasks
│       │
│       ├── encoder/
│       │   ├── mod.rs
│       │   ├── video_encoder.rs        ← H.265 FFmpeg encoder
│       │   ├── segment_manager.rs      ← Rolling .bbv segments; retention prune
│       │   ├── keyframe.rs             ← KeyframeIndex; seek logic
│       │   └── thumbnail.rs            ← JPEG thumb generation; encrypt; store
│       │
│       ├── crypto/
│       │   ├── mod.rs
│       │   ├── key_manager.rs          ← Master key; segment key wrap/unwrap
│       │   ├── encryptor.rs            ← AES-256-GCM encrypt primitives
│       │   ├── decryptor.rs            ← AES-256-GCM decrypt primitives
│       │   ├── kdf.rs                  ← Argon2id key derivation
│       │   └── keychain.rs             ← OS keychain interface (trait + impls)
│       │
│       ├── storage/
│       │   ├── mod.rs
│       │   ├── db.rs                   ← SQLCipher connection pool (sqlx)
│       │   ├── migrations/
│       │   │   ├── 001_initial.sql
│       │   │   ├── 002_indexes.sql
│       │   │   └── 003_fts_triggers.sql
│       │   ├── segment_repo.rs
│       │   ├── event_repo.rs
│       │   ├── window_repo.rs
│       │   ├── thumbnail_repo.rs
│       │   ├── ocr_repo.rs
│       │   └── settings_repo.rs
│       │
│       ├── ocr/
│       │   ├── mod.rs
│       │   ├── engine.rs               ← OcrEngine; Tesseract wrapper
│       │   ├── preprocessor.rs         ← Grayscale, denoise, upscale
│       │   ├── scheduler.rs            ← OCR cadence; change detection
│       │   └── indexer.rs              ← FTS5 insert; trigger management
│       │
│       ├── search/
│       │   ├── mod.rs
│       │   ├── engine.rs               ← SearchEngine; query routing
│       │   ├── text_search.rs          ← FTS5 full-text queries
│       │   ├── metadata_search.rs      ← Time/app/window filter queries
│       │   └── result.rs               ← SearchResult types
│       │
│       ├── replay/
│       │   ├── mod.rs
│       │   ├── player.rs               ← ReplaySession; seek; frame emit
│       │   ├── decoder.rs              ← H.265 decode via FFmpeg
│       │   ├── event_overlay.rs        ← Query+attach events to frames
│       │   └── exporter.rs             ← Export clip to mp4/mkv
│       │
│       ├── config/
│       │   ├── mod.rs
│       │   ├── settings.rs             ← AppSettings struct + TOML loader
│       │   ├── defaults.rs
│       │   └── validator.rs
│       │
│       ├── system/
│       │   ├── mod.rs
│       │   ├── tray.rs                 ← System tray menu + handlers
│       │   ├── autostart.rs            ← Launch-at-login registration
│       │   ├── permissions.rs          ← Platform permission checks
│       │   └── resource_monitor.rs     ← CPU/disk usage; throttle signals
│       │
│       └── commands/
│           ├── mod.rs
│           ├── capture_commands.rs
│           ├── search_commands.rs
│           ├── replay_commands.rs
│           ├── settings_commands.rs
│           ├── vault_commands.rs
│           └── system_commands.rs
│
├── src/                                ← Svelte frontend
│   ├── app.html
│   ├── app.css
│   ├── app.d.ts
│   ├── routes/
│   │   ├── +layout.svelte
│   │   ├── +page.svelte                ← Timeline (default view)
│   │   ├── search/+page.svelte
│   │   ├── replay/+page.svelte
│   │   └── settings/+page.svelte
│   ├── lib/
│   │   ├── components/
│   │   │   ├── Timeline/
│   │   │   │   ├── Timeline.svelte
│   │   │   │   ├── TimelineGroup.svelte
│   │   │   │   └── ThumbnailCard.svelte
│   │   │   ├── VideoPlayer/
│   │   │   │   ├── VideoPlayer.svelte
│   │   │   │   ├── Scrubber.svelte
│   │   │   │   └── EventOverlay.svelte
│   │   │   ├── Search/
│   │   │   │   ├── SearchBar.svelte
│   │   │   │   └── SearchResultCard.svelte
│   │   │   ├── Settings/
│   │   │   │   ├── SettingsPanel.svelte
│   │   │   │   ├── CaptureSettings.svelte
│   │   │   │   ├── PrivacySettings.svelte
│   │   │   │   └── VaultSettings.svelte
│   │   │   └── UI/
│   │   │       ├── TrayIcon.svelte
│   │   │       ├── Toast.svelte
│   │   │       └── PermissionPrompt.svelte
│   │   ├── stores/
│   │   │   ├── timeline.ts
│   │   │   ├── search.ts
│   │   │   ├── player.ts
│   │   │   ├── capture.ts
│   │   │   └── settings.ts
│   │   ├── api/
│   │   │   ├── client.ts               ← All invoke() wrappers
│   │   │   └── events.ts               ← All listen() wrappers
│   │   └── types/
│   │       └── index.ts
│   └── static/
│       └── favicon.png
│
├── scripts/
│   ├── install.sh                      ← macOS/Linux installer
│   ├── install.ps1                     ← Windows installer
│   └── check_permissions.sh
│
├── tests/
│   ├── unit/
│   │   ├── crypto_test.rs
│   │   ├── encoder_test.rs
│   │   ├── ocr_test.rs
│   │   └── search_test.rs
│   └── integration/
│       ├── capture_encode_test.rs
│       ├── search_replay_test.rs
│       └── retention_test.rs
│
├── .blackbox/                          ← Gitignored runtime data
│   ├── segments/                       ← .bbv encrypted video segments
│   ├── thumbs/                         ← .bbt encrypted thumbnails
│   ├── blackbox.db                     ← SQLCipher database
│   └── config.toml
│
├── Makefile
├── Cargo.toml                          ← Workspace root
├── package.json                        ← SvelteKit + Tauri
├── svelte.config.js
├── vite.config.ts
└── README.md

8. Configuration System
8.1 Config File (TOML)

Stored at ~/.blackbox/config.toml:

toml

[capture]
fps = 5
resolution = "1280x720"          # "native", "1920x1080", "1280x720", "854x480"
quality_crf = 28                 # H.265 CRF; 18-28 typical; higher = smaller + lower quality
monitors = "primary"             # "primary", "all", "mosaic", ["monitor:0", "monitor:1"]
capture_audio = false
codec = "h265"                   # "h265" | "h264" (h264 for compatibility fallback)
encoder_preset = "veryfast"      # ffmpeg preset: ultrafast..veryslow

[buffer]
max_duration_hours = 48
max_disk_gb = 5.0
storage_path = "~/.blackbox"
auto_prune = true
prune_check_interval_secs = 60

[ocr]
enabled = true
interval_secs = 5.0
languages = ["eng"]
min_confidence = 0.60
skip_unchanged_frames = true
max_queue_size = 100             # Drop OCR jobs if queue exceeds this

[privacy]
skip_apps = [
    "1Password",
    "Bitwarden",
    "KeePass",
    "LastPass",
    "Dashlane",
]
skip_title_patterns = [
    "Private.*Browsing",
    "Incognito",
    "InPrivate",
    ".*\\.onion",
]
redact_keystrokes_in_apps = [
    "Terminal",
    "iTerm",
    "Windows Terminal",
    "1Password",
    "Bitwarden",
    "sudo",
]
capture_raw_keystrokes = false   # Off by default; only shortcuts captured
skip_private_browsing = true

[encryption]
algorithm = "aes_256_gcm"       # Only option in v1; AES-256-GCM
kdf = "argon2id"
argon2_memory_kb = 65536        # 64 MB
argon2_iterations = 3
argon2_parallelism = 1
keychain_service = "BlackBoxLogger"
auto_lock_minutes = 60
encrypt_thumbnails = true
encrypt_ocr_text = false         # OCR text stored plaintext in SQLCipher DB

[ui]
start_minimized = true
launch_at_login = false
theme = "system"                 # "system" | "light" | "dark"
show_status_in_tray = true
default_view = "timeline"

[performance]
max_cpu_percent = 10
pause_on_battery = true
encoder_thread_priority = "low"
ocr_thread_priority = "background"

8.2 Config Loader

Rust

// config/settings.rs

use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AppConfig {
    pub capture:     CaptureConfig,
    pub buffer:      BufferConfig,
    pub ocr:         OcrConfig,
    pub privacy:     PrivacyConfig,
    pub encryption:  EncryptionConfig,
    pub ui:          UiConfig,
    pub performance: PerformanceConfig,
}

impl AppConfig {
    pub fn load() -> Result<Self> {
        let path = Self::default_path();
        if !path.exists() {
            let config = Self::default();
            config.save()?;
            return Ok(config);
        }
        let content = std::fs::read_to_string(&path)?;
        let config: Self = toml::from_str(&content)?;
        config.validate()?;
        Ok(config)
    }

    pub fn save(&self) -> Result<()> {
        let path = Self::default_path();
        std::fs::create_dir_all(path.parent().unwrap())?;
        let content = toml::to_string_pretty(self)?;
        std::fs::write(&path, content)?;
        Ok(())
    }

    pub fn default_path() -> PathBuf {
        let home = dirs::home_dir().expect("no home dir");
        home.join(".blackbox").join("config.toml")
    }

    fn validate(&self) -> Result<()> {
        if self.capture.fps == 0 || self.capture.fps > 30 {
            return Err(ConfigError::InvalidFps(self.capture.fps));
        }
        if self.buffer.max_disk_gb < 0.5 {
            return Err(ConfigError::DiskCapTooSmall);
        }
        if self.encryption.argon2_memory_kb < 8192 {
            return Err(ConfigError::KdfMemoryTooLow);
        }
        Ok(())
    }
}

8.3 Hot-Reload vs Restart-Required Settings
Setting	Hot-Reload Safe?	Notes
privacy.skip_apps	✅ Yes	Applied on next capture cycle
privacy.skip_title_patterns	✅ Yes	Applied on next capture cycle
ocr.interval_secs	✅ Yes	OCR scheduler reads dynamically
buffer.max_disk_gb	✅ Yes	Cleaner reads on next prune cycle
capture.fps	✅ Yes	Ticker interval updated
capture.resolution	⚠️ Restart	Encoder and scaler must be recreated
capture.codec	⚠️ Restart	Encoder must be recreated
encryption.algorithm	⚠️ Restart	Full re-key required
capture.monitors	⚠️ Restart	Capture tasks must restart
encryption.auto_lock_minutes	✅ Yes	Timer resets on next tick
9. Cryptography Deep Dive
9.1 Threat Model (Crypto Scope)
Threat	Mitigation
Attacker reads disk while machine is off	AES-256-GCM encryption of all artifacts; without master key, data is computationally useless
Attacker reads disk while vault is locked	Master key wiped from memory on lock; same as above
Attacker modifies encrypted segment file	AES-GCM provides integrity (tag verification fails on tamper)
Attacker reads SQLite database	SQLCipher full-database encryption; key derived from master key
Password brute force	Argon2id KDF with high memory cost; salt per vault
Malware reads process memory	Master key held in-memory only while unlocked; mlock() where possible
Pruned segment forensic recovery	Crypto-erase: key material deleted first; remnant ciphertext is useless
Thumbnail data leakage	Thumbnails encrypted with per-thumbnail AEAD keys or master-key-derived keys
9.2 Master Key Derivation

Rust

// crypto/kdf.rs

use argon2::{Argon2, Algorithm, Version, Params};

pub fn argon2id_derive(
    password: &[u8],
    salt: &[u8],
    memory_kb: u32,
    iterations: u32,
    parallelism: u32,
) -> Result<[u8; 32]> {
    let params = Params::new(memory_kb, iterations, parallelism, Some(32))
        .map_err(|e| KdfError::InvalidParams(e.to_string()))?;

    let argon2 = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);

    let mut output = [0u8; 32];
    argon2.hash_password_into(password, salt, &mut output)
        .map_err(|e| KdfError::DerivationFailed(e.to_string()))?;

    Ok(output)
}

9.3 AES-256-GCM Primitives

Rust

// crypto/encryptor.rs

use aes_gcm::{Aes256Gcm, Key, Nonce, aead::{Aead, KeyInit, Payload}};

/// Encrypt plaintext with AES-256-GCM.
/// Returns (ciphertext, nonce, tag) — tag appended to ciphertext by aes-gcm crate.
pub fn aes_gcm_encrypt(
    key: &[u8; 32],
    nonce: &[u8; 12],
    plaintext: &[u8],
) -> Result<(Vec<u8>, [u8; 12], [u8; 16])> {
    let cipher = Aes256Gcm::new(Key::<Aes256Gcm>::from_slice(key));
    let nonce = Nonce::from_slice(nonce);

    // aes-gcm appends 16-byte tag to ciphertext
    let ciphertext_with_tag = cipher
        .encrypt(nonce, plaintext)
        .map_err(|_| CryptoError::EncryptionFailed)?;

    let ciphertext = ciphertext_with_tag[..ciphertext_with_tag.len() - 16].to_vec();
    let mut tag = [0u8; 16];
    tag.copy_from_slice(&ciphertext_with_tag[ciphertext_with_tag.len() - 16..]);

    Ok((ciphertext, *nonce.as_ref(), tag))
}

/// Decrypt ciphertext with AES-256-GCM. Verifies authentication tag.
/// Returns an error if the tag is wrong (tampered data).
pub fn aes_gcm_decrypt(
    key: &[u8; 32],
    nonce: &[u8; 12],
    ciphertext: &[u8],
    tag: &[u8; 16],
) -> Result<Vec<u8>> {
    let cipher = Aes256Gcm::new(Key::<Aes256Gcm>::from_slice(key));
    let nonce = Nonce::from_slice(nonce);

    // Reassemble ciphertext + tag for aes-gcm crate
    let mut ct_with_tag = ciphertext.to_vec();
    ct_with_tag.extend_from_slice(tag);

    cipher.decrypt(nonce, ct_with_tag.as_slice())
        .map_err(|_| CryptoError::DecryptionFailed)
}

9.4 Per-Chunk Nonce Derivation

Each chunk within a segment uses a unique nonce derived from the segment's nonce_base and the chunk counter:

Rust

// crypto/encryptor.rs

/// Derive a unique 12-byte nonce for chunk `counter` within a segment.
/// Uses: nonce_base (12 bytes) XOR counter (8 bytes, little-endian, padded to 12)
pub fn derive_chunk_nonce(nonce_base: &[u8; 12], counter: u64) -> [u8; 12] {
    let mut nonce = *nonce_base;
    let counter_bytes = counter.to_le_bytes();  // 8 bytes
    for (i, b) in counter_bytes.iter().enumerate() {
        nonce[i] ^= b;
    }
    nonce
}

Security property: As long as nonce_base is unique per segment (generated via rand::random()), all chunk nonces within that segment are unique, satisfying AES-GCM's nonce-uniqueness requirement.
9.5 OS Keychain Backends

Rust

// crypto/keychain.rs

pub trait KeychainBackend: Send + Sync {
    fn store(&self, key_id: &str, secret: &[u8]) -> Result<()>;
    fn load(&self, key_id: &str) -> Result<Vec<u8>>;
    fn delete(&self, key_id: &str) -> Result<()>;
}

#[cfg(target_os = "macos")]
pub struct MacOsKeychain;
#[cfg(target_os = "macos")]
impl KeychainBackend for MacOsKeychain {
    fn store(&self, key_id: &str, secret: &[u8]) -> Result<()> {
        // keychain crate: SecItemAdd with kSecClassGenericPassword
    }
    fn load(&self, key_id: &str) -> Result<Vec<u8>> {
        // SecItemCopyMatching
    }
    fn delete(&self, key_id: &str) -> Result<()> {
        // SecItemDelete
    }
}

#[cfg(target_os = "windows")]
pub struct WindowsCredentialManager;
#[cfg(target_os = "windows")]
impl KeychainBackend for WindowsCredentialManager {
    // CredWrite / CredRead / CredDelete
}

#[cfg(target_os = "linux")]
pub struct LinuxSecretService;
#[cfg(target_os = "linux")]
impl KeychainBackend for LinuxSecretService {
    // libsecret / secret-service DBus API
    // Fallback: encrypted file in ~/.blackbox/ if libsecret unavailable
}

10. Segment File Format (.bbv) Specification
10.1 File Layout

text

┌─────────────────────────────────────────────────────┐
│                   HEADER (plaintext)                 │
│  Magic:            "BBLV" (4 bytes)                  │
│  Version:          u16 (little-endian)               │
│  Header Length:    u32 (length of JSON header block) │
│  Header JSON:      UTF-8 JSON (see 10.2)             │
├─────────────────────────────────────────────────────┤
│                   CHUNK 0                            │
│  Counter:          u64 LE (= 0)                      │
│  Ciphertext Len:   u32 LE                            │
│  Ciphertext:       bytes[CiphertextLen]              │
│  GCM Tag:          16 bytes                          │
├─────────────────────────────────────────────────────┤
│                   CHUNK 1                            │
│  ...same structure...                                │
├─────────────────────────────────────────────────────┤
│                   ...                                │
├─────────────────────────────────────────────────────┤
│                   FOOTER                             │
│  Footer Magic:     "BBLF" (4 bytes)                  │
│  Keyframe Index:   u32 LE (length) + JSON bytes      │
│  Segment Hash:     32 bytes (SHA-256 of all chunks)  │
└─────────────────────────────────────────────────────┘

10.2 Header JSON Schema

JSON

{
  "magic": "BBLV",
  "version": 1,
  "segment_id": "<uuid>",
  "capture_stream_id": "monitor:0",
  "created_at_utc": 1748391600000,
  "codec": "h265",
  "width": 1280,
  "height": 720,
  "fps": 5.0,
  "encryption_alg": "AES_GCM_256",
  "nonce_base_hex": "aabbccddeeff00112233445566",
  "chunk_size_bytes": 65536,
  "blackbox_version": "1.0.0"
}

10.3 Rust Structs

Rust

// encoder/segment_file.rs

pub const FILE_MAGIC: &[u8; 4] = b"BBLV";
pub const FOOTER_MAGIC: &[u8; 4] = b"BBLF";
pub const FORMAT_VERSION: u16 = 1;

#[derive(Debug, Serialize, Deserialize)]
pub struct SegmentHeader {
    pub magic: String,
    pub version: u16,
    pub segment_id: Uuid,
    pub capture_stream_id: String,
    pub created_at_utc: i64,
    pub codec: String,
    pub width: u32,
    pub height: u32,
    pub fps: f64,
    pub encryption_alg: String,
    pub nonce_base_hex: String,
    pub chunk_size_bytes: u32,
    pub blackbox_version: String,
}

pub struct Chunk {
    pub counter: u64,
    pub ciphertext: Vec<u8>,
    pub tag: [u8; 16],
    pub nonce: [u8; 12],  // Derived; not stored (computable from base+counter)
}

pub struct SegmentFileWriter {
    file: BufWriter<File>,
    path: PathBuf,
    position: u64,
}

impl SegmentFileWriter {
    pub fn new(path: PathBuf, header: &SegmentHeader) -> Result<Self> {
        let mut file = BufWriter::new(File::create(&path)?);

        // Write magic
        file.write_all(FILE_MAGIC)?;
        // Write version
        file.write_all(&FORMAT_VERSION.to_le_bytes())?;
        // Serialize header JSON
        let header_json = serde_json::to_vec(header)?;
        file.write_all(&(header_json.len() as u32).to_le_bytes())?;
        file.write_all(&header_json)?;

        let position = (4 + 2 + 4 + header_json.len()) as u64;
        Ok(SegmentFileWriter { file, path, position })
    }

    pub fn write_chunk(&mut self, chunk: Chunk) -> Result<()> {
        self.file.write_all(&chunk.counter.to_le_bytes())?;
        self.file.write_all(&(chunk.ciphertext.len() as u32).to_le_bytes())?;
        self.file.write_all(&chunk.ciphertext)?;
        self.file.write_all(&chunk.tag)?;
        self.position += 8 + 4 + chunk.ciphertext.len() as u64 + 16;
        Ok(())
    }

    pub fn write_footer(&mut self, keyframes: &[KeyframeEntry]) -> Result<()> {
        self.file.write_all(FOOTER_MAGIC)?;
        let kf_json = serde_json::to_vec(keyframes)?;
        self.file.write_all(&(kf_json.len() as u32).to_le_bytes())?;
        self.file.write_all(&kf_json)?;
        Ok(())
    }

    pub fn position(&self) -> u64 { self.position }
    pub fn path(&self) -> &Path { &self.path }
}

11. OCR Pipeline Deep Dive
11.1 Change Detection Algorithm

Rust

// ocr/scheduler.rs

pub struct FrameChangeDetector {
    last_phash: Option<u64>,
    similarity_threshold: f32,  // Default 0.90 = skip if 90%+ similar
}

impl FrameChangeDetector {
    /// Returns true if the frame has changed enough to warrant OCR
    pub fn has_changed(&mut self, frame: &VideoFrame) -> bool {
        // Compute perceptual hash (pHash: 8x8 DCT-based)
        let hash = perceptual_hash_8x8(
            &frame.data,
            frame.width, frame.height,
        );

        match self.last_phash {
            None => {
                self.last_phash = Some(hash);
                true  // Always OCR first frame
            }
            Some(prev) => {
                let similarity = hamming_similarity(prev, hash);
                if similarity < self.similarity_threshold {
                    self.last_phash = Some(hash);
                    true
                } else {
                    false  // Skip: too similar to last OCR'd frame
                }
            }
        }
    }
}

/// Perceptual hash: downsample to 8x8 grayscale, DCT, median threshold
fn perceptual_hash_8x8(data: &[u8], w: u32, h: u32) -> u64 {
    let small = bilinear_resize_gray(data, w, h, 8, 8);
    let dct = dct_8x8(&small);
    let median = median_f32(&dct[..32]);  // Top-left 32 coefficients
    let mut hash: u64 = 0;
    for (i, &v) in dct[..64].iter().enumerate() {
        if v > median {
            hash |= 1 << i;
        }
    }
    hash
}

fn hamming_similarity(a: u64, b: u64) -> f32 {
    let diff = (a ^ b).count_ones();
    1.0 - (diff as f32 / 64.0)
}

11.2 Tesseract Integration

Rust

// ocr/engine.rs

use tesseract::Tesseract;

pub struct TesseractInstance {
    api: Tesseract,
}

impl TesseractInstance {
    pub fn new(languages: &[String]) -> Result<Self> {
        let lang = languages.join("+");
        let api = Tesseract::new(None, Some(&lang))
            .map_err(|e| OcrError::InitFailed(e.to_string()))?;
        // Set to LSTM only (more accurate than legacy)
        api.set_variable("tessedit_ocr_engine_mode", "1")?;
        // Assume single uniform block of text (no complex layout)
        api.set_variable("tessedit_pageseg_mode", "6")?;
        Ok(TesseractInstance { api })
    }

    pub fn recognize(&mut self, gray_image: &GrayImage) -> Result<TesseractOutput> {
        self.api.set_image_from_mem(
            gray_image.as_raw(),
            gray_image.width() as i32,
            gray_image.height() as i32,
            1,  // bytes per pixel (grayscale)
            gray_image.width() as i32,
        )?;

        let text = self.api.get_utf8_text()
            .map_err(|e| OcrError::RecognitionFailed(e.to_string()))?;
        let confidence = self.api.mean_text_conf() as f32 / 100.0;

        Ok(TesseractOutput { text, mean_confidence: confidence })
    }
}

11.3 OCR Scheduler Task

Rust

// ocr/scheduler.rs

pub struct OcrScheduler {
    engine: OcrEngine,
    detector: FrameChangeDetector,
    config: OcrConfig,
    db: Arc<Db>,
    queue: mpsc::Receiver<OcrJob>,
    last_ocr_time: Instant,
}

impl OcrScheduler {
    pub async fn run(&mut self) {
        loop {
            match self.queue.try_recv() {
                Ok(job) => {
                    // Enforce minimum interval
                    if self.last_ocr_time.elapsed().as_secs_f64() < self.config.interval_secs {
                        continue;  // Too soon; skip this job
                    }

                    // Check frame changed
                    if !self.detector.has_changed(&job.frame) {
                        continue;
                    }

                    // Run OCR
                    match self.engine.process_frame(&job.frame, job.context).await {
                        Ok(Some(result)) => {
                            if let Err(e) = self.db.insert_ocr_frame(result).await {
                                tracing::warn!(error = ?e, "OCR insert failed");
                            }
                        }
                        Ok(None) => {}  // Below confidence threshold; skip
                        Err(e) => {
                            tracing::warn!(error = ?e, "OCR processing error");
                        }
                    }

                    self.last_ocr_time = Instant::now();
                }
                Err(mpsc::error::TryRecvError::Empty) => {
                    tokio::time::sleep(Duration::from_millis(100)).await;
                }
                Err(mpsc::error::TryRecvError::Disconnected) => break,
            }
        }
    }
}

12. Search System Deep Dive
12.1 Query Plan Decision Tree

text

User submits SearchQuery
        │
        ├── has text? ──Yes──► FTS5 MATCH query
        │                       + JOIN ocr_frames → windows → thumbnails
        │                       + optional time/app filters in WHERE
        │                       + ORDER BY bm25 score ASC
        │
        └── No text ──────────► Metadata-only query
                                SELECT from thumbnails
                                + JOIN windows
                                + WHERE time range, app name, window title
                                + ORDER BY timestamp DESC

12.2 FTS5 Query Construction

Rust

// search/text_search.rs

pub async fn fts5_search(
    db: &Db,
    query: &SearchQuery,
    limit: i64,
    offset: i64,
) -> Result<Vec<SearchResult>> {
    let text = query.text.as_deref().unwrap_or("");

    // Sanitize FTS5 query text to prevent injection
    let sanitized = sanitize_fts5_query(text);

    let sql = r#"
        SELECT
            of.id,
            bm25(ocr_fts) AS bm25_score,
            of.timestamp,
            of.segment_id,
            of.pts_ms,
            of.capture_stream_id,
            highlight(ocr_fts, 0, '<b>', '</b>') AS matched_text,
            snippet(ocr_fts, 0, '<b>', '</b>', ' ... ', 32) AS context_text,
            COALESCE(w.app_name, '') AS app_name,
            COALESCE(w.window_title, '') AS window_title,
            w.id AS window_id,
            t.id AS thumb_id,
            t.path AS thumb_path,
            t.nonce AS thumb_nonce,
            t.tag AS thumb_tag
        FROM ocr_fts
        INNER JOIN ocr_frames of ON ocr_fts.rowid = of.id
        LEFT JOIN windows w ON of.window_id = w.id
        LEFT JOIN thumbnails t ON t.segment_id = of.segment_id
            AND t.timestamp = (
                SELECT timestamp FROM thumbnails
                WHERE segment_id = of.segment_id
                ORDER BY ABS(timestamp - of.timestamp) ASC
                LIMIT 1
            )
        WHERE ocr_fts MATCH ?
        AND (? IS NULL OR of.timestamp >= ?)
        AND (? IS NULL OR of.timestamp <= ?)
        AND (? IS NULL OR LOWER(w.app_name) LIKE '%' || LOWER(?) || '%')
        ORDER BY bm25_score ASC
        LIMIT ? OFFSET ?
    "#;

    let app_filter = query.apps.as_ref().and_then(|a| a.first()).cloned();

    db.query_map(sql, params![
        sanitized,
        query.start_time, query.start_time,
        query.end_time, query.end_time,
        app_filter.clone(), app_filter,
        limit, offset,
    ], map_row_to_result).await
}

fn sanitize_fts5_query(input: &str) -> String {
    // Wrap bare terms in quotes; allow AND, OR, NOT, phrase queries
    // Strip characters that would break FTS5 syntax
    input
        .replace('"', "'")    // Replace double quotes to avoid FTS5 syntax break
        .trim()
        .to_string()
}

12.3 Result Deduplication

Rust

// search/engine.rs

pub fn deduplicate_results(results: Vec<SearchResult>, bucket_ms: i64) -> Vec<SearchResult> {
    let mut seen: HashMap<(String, i64), SearchResult> = HashMap::new();

    for result in results {
        // Time bucket: round timestamp to nearest N ms bucket
        let bucket_key = result.timestamp / bucket_ms;
        let key = (result.capture_stream_id.clone(), bucket_key);

        match seen.entry(key) {
            Entry::Vacant(e) => { e.insert(result); }
            Entry::Occupied(mut e) => {
                // Keep result with better (more negative) bm25 score
                if result.score < e.get().score {
                    e.insert(result);
                }
            }
        }
    }

    let mut deduped: Vec<SearchResult> = seen.into_values().collect();
    deduped.sort_by(|a, b| a.score.partial_cmp(&b.score).unwrap_or(Ordering::Equal));
    deduped
}

13. Replay Engine Deep Dive
13.1 Session State Machine

text

    ┌─────────┐
    │  IDLE   │
    └────┬────┘
         │ create_session(timestamp)
         ▼
    ┌─────────┐          replay_seek(ts)
    │ SEEKING │◄──────────────────────────┐
    └────┬────┘                           │
         │ seek complete                  │
         ▼                               │
    ┌─────────┐  replay_play()  ┌──────────────┐
    │ PAUSED  │────────────────►│   PLAYING    │
    └────┬────┘                 └──────┬───────┘
         │                            │ end of segment
         │                            │ → SEEKING next segment
         │ replay_pause()             │
         └────────────────────────────┘
         │ replay_close()
         ▼
    ┌─────────┐
    │ CLOSED  │
    └─────────┘

13.2 Cross-Segment Playback

Rust

// replay/player.rs

impl ReplayEngine {
    pub async fn play_continuous(
        &self,
        session: &mut ReplaySession,
        app_handle: &tauri::AppHandle,
    ) -> Result<()> {
        let target_frame_ms = 1000 / 5;  // 5 FPS → 200ms per frame
        let mut current_ts = session.current_timestamp;

        loop {
            if session.state == ReplayState::Paused { break; }
            if session.state == ReplayState::Closed { break; }

            let frame_start = Instant::now();

            // Find segment for current timestamp
            let segment = self.db
                .find_segment_at(current_ts, &session.capture_stream_id)
                .await?;

            match segment {
                None => {
                    // Gap in recording (paused/skipped); advance and try again
                    current_ts += 1000;
                    continue;
                }
                Some(seg) => {
                    let key = self.key_manager.unwrap_segment_key(&seg.wrapped_key).await?;
                    let frame = self.decode_frame_at(&seg, &key, current_ts).await?;

                    // Query events for overlay
                    let events = self.db.get_events_in_range(current_ts - 250, current_ts + 250).await?;

                    let replay_frame = ReplayFrame {
                        session_id: session.session_id,
                        timestamp: current_ts,
                        pts_ms: frame.pts_ms,
                        width: frame.width,
                        height: frame.height,
                        data: frame.rgba_data,
                        events: events.into_iter().map(Into::into).collect(),
                    };

                    app_handle.emit_all("bb/replay_frame", &replay_frame)?;
                    current_ts += target_frame_ms;
                    session.current_timestamp = current_ts;
                }
            }

            // Maintain playback FPS
            let elapsed = frame_start.elapsed();
            let target = Duration::from_millis(target_frame_ms as u64);
            if elapsed < target {
                tokio::time::sleep(target - elapsed).await;
            }
        }

        Ok(())
    }
}

13.3 Event Overlay Rendering (Frontend)

svelte

<!-- src/lib/components/VideoPlayer/EventOverlay.svelte -->

<script lang="ts">
  import type { OverlayEvent } from '$lib/types';
  import { currentFrame } from '$lib/stores/player';

  $: events = $currentFrame?.events ?? [];
  $: clickEvents = events.filter(e => e.event_type === 'mouse_click');
  $: shortcutEvents = events.filter(e => e.event_type === 'shortcut');
  $: windowEvents = events.filter(e => e.event_type === 'window_focus');
</script>

<div class="overlay absolute inset-0 pointer-events-none">
  <!-- Mouse click indicators -->
  {#each clickEvents as ev}
    <div
      class="click-ring absolute w-8 h-8 rounded-full border-2 border-yellow-400 animate-ping"
      style="left: {ev.x}px; top: {ev.y}px; transform: translate(-50%, -50%);"
    />
  {/each}

  <!-- Shortcut banners -->
  {#each shortcutEvents as ev}
    <div class="shortcut-banner absolute bottom-4 left-4 bg-black/70 text-white
                text-sm px-3 py-1 rounded-md font-mono">
      ⌨ {ev.description}
    </div>
  {/each}

  <!-- Window change banners -->
  {#each windowEvents as ev}
    <div class="window-banner absolute top-2 left-2 right-2 bg-blue-500/80 text-white
                text-xs px-2 py-1 rounded text-center">
      🪟 {ev.description}
    </div>
  {/each}
</div>

14. Privacy & Redaction Module
14.1 Skip Logic (Full Flow)

Rust

// capture/manager.rs

pub struct PrivacyGuard {
    config: PrivacyConfig,
    current_window: Arc<RwLock<Option<WindowInfo>>>,
}

pub enum CaptureDecision {
    Capture,
    Skip { reason: SkipReason },
}

pub enum SkipReason {
    SkippedApp(String),
    SkippedTitlePattern(String),
    PrivateBrowsing,
    VaultLocked,
    ManualPause,
}

impl PrivacyGuard {
    pub fn should_capture(&self) -> CaptureDecision {
        let window = self.current_window.read().unwrap();

        if let Some(ref win) = *window {
            // Check skip apps
            for app in &self.config.skip_apps {
                if win.app_name.to_lowercase().contains(&app.to_lowercase()) {
                    return CaptureDecision::Skip {
                        reason: SkipReason::SkippedApp(win.app_name.clone()),
                    };
                }
            }

            // Check title patterns (regex)
            for pattern in &self.config.skip_title_patterns {
                if let Ok(re) = Regex::new(pattern) {
                    if re.is_match(&win.window_title) {
                        return CaptureDecision::Skip {
                            reason: SkipReason::SkippedTitlePattern(pattern.clone()),
                        };
                    }
                }
            }

            // Private browsing heuristic
            if self.config.skip_private_browsing && win.is_browser {
                let title = win.window_title.to_lowercase();
                if title.contains("private") || title.contains("incognito")
                    || title.contains("inprivate")
                {
                    return CaptureDecision::Skip {
                        reason: SkipReason::PrivateBrowsing,
                    };
                }
            }
        }

        CaptureDecision::Capture
    }
}

14.2 Known Limitations of Title-Pattern Matching
Limitation	Impact	Mitigation
Browser/language differences	"Incognito" is English; other locales differ	Regex list is user-configurable; document known variants
Custom window titles	User or app can set any title	User can add custom patterns in settings
Banking sites in normal mode	Won't be skipped without a rule	User can add domain/title rules
Obfuscated app names	1pword wouldn't match 1Password	Regex matching; user configures
15. Security Model & Threat Boundaries
15.1 Trust Zones

text

┌───────────────────────────────────────────────────────┐
│  FULLY TRUSTED (Rust process, OS keychain)             │
│  - Capture tasks                                       │
│  - Encoder + segment writer                            │
│  - Crypto / key manager                                │
│  - SQLCipher database                                  │
│  - Encrypted .bbv / .bbt artifacts on disk             │
└───────────────────────────────────┬───────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────┐
│  SEMI-TRUSTED (Tauri WebView)                          │
│  - Svelte frontend (runs in Tauri-sandboxed WebView)   │
│  - Can only call explicitly registered #[tauri::command]│
│  - Cannot access filesystem directly                   │
│  - Cannot make arbitrary network requests              │
└───────────────────────────────────┬───────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────┐
│  UNTRUSTED (screen content, clipboard, input)          │
│  - Whatever is on screen (websites, files, etc.)       │
│  - Clipboard contents                                  │
│  - OCR output (user-generated text from screen)        │
└───────────────────────────────────────────────────────┘

15.2 Tauri Security Configuration

JSON

// src-tauri/tauri.conf.json (security-relevant section)

{
  "tauri": {
    "security": {
      "csp": "default-src 'self'; script-src 'self'; connect-src ipc: http://ipc.localhost",
      "dangerousDisableAssetCspModification": false,
      "freezePrototype": true
    },
    "allowlist": {
      "all": false,
      "fs": {
        "all": false,
        "readFile": false,
        "writeFile": false,
        "readDir": false
      },
      "dialog": {
        "open": true,
        "save": true
      },
      "notification": {
        "all": true
      },
      "window": {
        "all": false,
        "close": true,
        "hide": true,
        "show": true,
        "minimize": true
      }
    }
  }
}

Key security properties enforced by this config:

    The WebView has no direct filesystem access; all file I/O goes through Rust commands
    Thumbnails and replay frames are decrypted in Rust and sent as base64/RGBA — never as raw file paths
    CSP blocks all external network requests from the UI
    All Tauri commands are explicitly allowlisted; there is no wildcard all: true

15.3 Memory Security

Rust

// crypto/key_manager.rs

use zeroize::Zeroize;

/// Master key is wrapped in a type that auto-zeros on drop
#[derive(Zeroize)]
#[zeroize(drop)]
struct MasterKey([u8; 32]);

pub struct KeyManager {
    master_key: Arc<Mutex<Option<MasterKey>>>,
    // ...
}

impl KeyManager {
    pub fn lock(&self) {
        let mut key = self.master_key.lock().unwrap();
        // Dropping the MasterKey zeroizes it via the Zeroize derive
        *key = None;
        // Additionally: try to advise OS to not swap this page
        // (platform-specific: mlock / VirtualLock)
    }
}

15.4 Threat Model Summary
Threat	Severity	Mitigation
Physical access to disk (machine off)	Critical	AES-256-GCM encrypted all artifacts; crypto-erase on prune
Disk access while vault locked	High	Master key wiped from memory on lock
Tampered segment files	Medium	GCM authentication tag detects any modification
Password brute force	Medium	Argon2id with 64MB memory cost; slow per-guess
Malware in same OS session reading memory	High	Minimize key residency time; zeroize on lock
SQLite DB exposed	High	SQLCipher full-database encryption
Thumbnail data leakage	Medium	Thumbnails encrypted as .bbt; never served raw
OCR leaking sensitive text	Medium	OCR text in SQLCipher DB; configurable redaction of apps
Clipboard leakage	High	Clipboard payload encrypted before storage
Unencrypted temp files	Medium	All temp decryption in-memory only; no intermediate files
16. Storage & Persistence
16.1 SQLCipher Connection

Rust

// storage/db.rs

use sqlx::sqlite::{SqliteConnectOptions, SqlitePool};
use std::str::FromStr;

pub struct Db {
    pool: SqlitePool,
}

impl Db {
    pub async fn new(path: &Path, db_key: &str) -> Result<Self> {
        // SQLCipher: pass encryption key via PRAGMA
        // The key is derived from the same master key material
        let db_url = format!("sqlite:{}", path.display());

        let options = SqliteConnectOptions::from_str(&db_url)?
            .pragma("key", db_key)              // SQLCipher encryption key
            .pragma("journal_mode", "WAL")       // Better concurrent read performance
            .pragma("busy_timeout", "5000")
            .pragma("synchronous", "NORMAL")
            .pragma("foreign_keys", "ON")
            .create_if_missing(true);

        let pool = SqlitePool::connect_with(options).await?;

        let db = Db { pool };
        db.run_migrations().await?;
        Ok(db)
    }

    async fn run_migrations(&self) -> Result<()> {
        sqlx::migrate!("./src/storage/migrations")
            .run(&self.pool)
            .await
            .map_err(|e| StorageError::MigrationFailed(e.to_string()))
    }
}

16.2 Retention & Pruning

Rust

// storage/cleaner.rs

pub struct RetentionCleaner {
    config: BufferConfig,
    db: Arc<Db>,
    storage_dir: PathBuf,
}

impl RetentionCleaner {
    pub async fn run_once(&self) -> Result<CleanupReport> {
        let mut report = CleanupReport::default();

        // Strategy 1: Prune by age
        let cutoff_ms = utc_now_ms()
            - (self.config.max_duration_hours * 3600.0 * 1000.0) as i64;

        let old_segments = self.db.get_segments_before(cutoff_ms).await?;
        for seg in &old_segments {
            self.crypto_erase_segment(seg).await?;
            report.segments_pruned += 1;
        }

        // Strategy 2: Prune by disk size (oldest first until under cap)
        let total_bytes = self.total_storage_bytes().await?;
        let max_bytes = (self.config.max_disk_gb * 1e9) as i64;

        if total_bytes > max_bytes {
            let over_by = total_bytes - max_bytes;
            let mut freed: i64 = 0;
            let oldest = self.db.get_oldest_segments(100).await?;

            for seg in oldest {
                if freed >= over_by { break; }
                let seg_size = seg.file_size_bytes.unwrap_or(0);
                self.crypto_erase_segment(&seg).await?;
                freed += seg_size;
                report.segments_pruned += 1;
            }
        }

        // Strategy 3: Clean up orphaned thumbnail files
        self.prune_orphan_thumbs().await?;

        Ok(report)
    }

    /// Crypto-erase: delete key material first, then file
    async fn crypto_erase_segment(&self, seg: &SegmentRecord) -> Result<()> {
        // 1. Delete segment key from DB (key material gone → ciphertext useless)
        self.db.delete_segment_wrapped_key(&seg.id).await?;

        // 2. Delete associated thumbnails from DB + disk
        let thumbs = self.db.get_thumbnails_for_segment(&seg.id).await?;
        for thumb in thumbs {
            std::fs::remove_file(&thumb.path).ok();
            self.db.delete_thumbnail(&thumb.id).await?;
        }

        // 3. Delete OCR entries (cascades via FK in DB)
        self.db.delete_ocr_for_segment(&seg.id).await?;

        // 4. Delete segment file
        std::fs::remove_file(&seg.path).ok();

        // 5. Delete segment record
        self.db.delete_segment(&seg.id).await?;

        Ok(())
    }
}

17. Logging, Observability & Debugging
17.1 Structured Logging

Rust

// main.rs

use tracing_subscriber::{fmt, EnvFilter, prelude::*};
use tracing_appender::rolling::{RollingFileAppender, Rotation};

fn init_logging(config: &LogConfig) {
    let file_appender = RollingFileAppender::new(
        Rotation::DAILY,
        &config.log_dir,
        "blackbox.log",
    );

    let subscriber = tracing_subscriber::registry()
        .with(EnvFilter::new(&config.level))
        .with(
            fmt::layer()
                .with_writer(file_appender)
                .json()
                .with_timer(fmt::time::UtcTime::rfc_3339())
                .with_target(true)
                .with_thread_ids(true)
        );

    tracing::subscriber::set_global_default(subscriber).unwrap();
}

Logging rules:

    NEVER log OCR text, clipboard contents, or raw keystrokes at any log level
    NEVER log encryption keys, nonces, or plaintext segment data
    Log capture decisions (skip/record) at DEBUG level with app name only
    Log segment finalization at INFO level with segment_id and duration
    Log encryption errors at ERROR level with error code only (not key material)
    Log performance metrics (dropped frames, OCR queue depth) at DEBUG level

17.2 Metrics Collection

Rust

// system/resource_monitor.rs

pub struct ResourceMonitor {
    config: PerformanceConfig,
    metrics: Arc<Metrics>,
    throttle_tx: mpsc::Sender<ThrottleCommand>,
}

pub struct Metrics {
    pub dropped_frames: AtomicU64,
    pub frames_encoded: AtomicU64,
    pub ocr_queue_depth: AtomicU64,
    pub ocr_jobs_processed: AtomicU64,
    pub segments_written: AtomicU64,
    pub total_disk_bytes: AtomicU64,
    pub vault_unlock_count: AtomicU64,
    pub search_queries: AtomicU64,
}

impl ResourceMonitor {
    pub async fn run(&self) {
        let mut interval = tokio::time::interval(Duration::from_secs(2));
        loop {
            interval.tick().await;

            let cpu_pct = get_process_cpu_percent();
            let disk_free_gb = get_disk_free_gb(&self.config.storage_path);

            if cpu_pct > self.config.max_cpu_percent as f64 {
                // Throttle OCR first (highest CPU consumer after encoder)
                self.throttle_tx.send(ThrottleCommand::ReduceOcrFrequency).await.ok();
                tracing::warn!(cpu_pct, "CPU budget exceeded; throttling OCR");
            }

            if disk_free_gb < 1.0 {
                // Emergency: pause recording + notify user
                self.throttle_tx.send(ThrottleCommand::PauseCapture).await.ok();
                tracing::error!(disk_free_gb, "Disk nearly full; pausing capture");
            }
        }
    }
}

18. Testing Strategy
18.1 Test Pyramid

text

                      ┌──────────────┐
                      │   E2E (5%)   │
                      │ Full capture │
                      │ → search →   │
                      │ replay flow  │
                      └──────┬───────┘
             ┌───────────────┴────────────────┐
             │     Integration Tests (25%)    │
             │  Encoder + Crypto + DB + OCR   │
             │  Segment prune, retention      │
             └───────────────┬────────────────┘
      ┌────────────────────── ┴────────────────────────┐
      │            Unit Tests (70%)                    │
      │  Crypto roundtrip, OCR postprocess, FTS5,      │
      │  change detection, key wrap/unwrap, BBV parse  │
      └────────────────────────────────────────────────┘

18.2 Unit Tests

Rust

// tests/unit/crypto_test.rs

#[test]
fn test_aes_gcm_roundtrip() {
    let key: [u8; 32] = rand::random();
    let nonce: [u8; 12] = rand::random();
    let plaintext = b"hello blackbox logger";

    let (ciphertext, n, tag) = aes_gcm_encrypt(&key, &nonce, plaintext).unwrap();
    assert_ne!(&ciphertext, plaintext);

    let decrypted = aes_gcm_decrypt(&key, &n, &ciphertext, &tag).unwrap();
    assert_eq!(&decrypted, plaintext);
}

#[test]
fn test_aes_gcm_tamper_detection() {
    let key: [u8; 32] = rand::random();
    let nonce: [u8; 12] = rand::random();
    let (mut ciphertext, n, tag) = aes_gcm_encrypt(&key, &nonce, b"sensitive").unwrap();

    // Tamper with ciphertext
    ciphertext[0] ^= 0xFF;

    let result = aes_gcm_decrypt(&key, &n, &ciphertext, &tag);
    assert!(result.is_err(), "Tampered ciphertext should fail authentication");
}

#[test]
fn test_argon2id_deterministic() {
    let password = b"correct horse battery staple";
    let salt = b"testsalt16bytess";
    let k1 = argon2id_derive(password, salt, 65536, 3, 1).unwrap();
    let k2 = argon2id_derive(password, salt, 65536, 3, 1).unwrap();
    assert_eq!(k1, k2, "Same inputs must produce same key");
}

#[test]
fn test_chunk_nonce_uniqueness() {
    let base: [u8; 12] = rand::random();
    let n0 = derive_chunk_nonce(&base, 0);
    let n1 = derive_chunk_nonce(&base, 1);
    let n255 = derive_chunk_nonce(&base, 255);
    assert_ne!(n0, n1);
    assert_ne!(n0, n255);
    assert_ne!(n1, n255);
}

// tests/unit/ocr_test.rs

#[test]
fn test_ocr_postprocess_min_words() {
    let engine = OcrEngine::new_test();
    let raw = TesseractOutput { text: "hi".into(), mean_confidence: 0.95 };
    let result = engine.postprocess(raw).unwrap();
    assert!(result.text.is_empty(), "Less than 3 words should be filtered");
}

#[test]
fn test_ocr_postprocess_low_confidence() {
    let engine = OcrEngine::new_test();
    let raw = TesseractOutput { text: "hello world there".into(), mean_confidence: 0.30 };
    let result = engine.postprocess(raw).unwrap();
    assert!(result.text.is_empty(), "Low confidence should be filtered");
}

#[test]
fn test_perceptual_hash_unchanged() {
    let frame1 = load_test_frame("tests/fixtures/screen1.png");
    let frame2 = load_test_frame("tests/fixtures/screen1.png");  // Same frame
    let h1 = perceptual_hash_8x8(&frame1.data, frame1.width, frame1.height);
    let h2 = perceptual_hash_8x8(&frame2.data, frame2.width, frame2.height);
    assert_eq!(h1, h2);
    assert!(hamming_similarity(h1, h2) > 0.95);
}

// tests/unit/search_test.rs

#[tokio::test]
async fn test_fts5_search_finds_term() {
    let db = Db::new_in_memory().await.unwrap();
    db.insert_ocr_frame(OcrFrameRecord {
        id: 1, timestamp: 1000, text: "forgot to save the document".into(),
        confidence: 0.85, ..Default::default()
    }).await.unwrap();

    let results = fts5_search(&db, &SearchQuery {
        text: Some("forgot to save".into()),
        ..Default::default()
    }, 10, 0).await.unwrap();

    assert!(!results.is_empty());
    assert!(results[0].matched_text.contains("forgot"));
}

18.3 Integration Tests

Rust

// tests/integration/capture_encode_test.rs

#[tokio::test]
async fn test_full_capture_encode_encrypt_store_cycle() {
    let dir = tempdir().unwrap();
    let config = test_config(dir.path());
    let db = Db::new(&dir.path().join("test.db"), "testkey").await.unwrap();
    let key_manager = KeyManager::new_test();
    key_manager.unlock("testpassword").await.unwrap();
    let segment_manager = SegmentManager::new(config.clone(), Arc::new(db.clone()), Arc::new(key_manager));

    // Feed synthetic frames
    let encoder = H265Encoder::new(config.capture.into()).unwrap();
    for i in 0..30 {  // 30 frames = 6 seconds at 5fps
        let frame = VideoFrame::synthetic_solid(1280, 720, i as u8, i as u8, 0);
        let packets = encoder.encode_frame(&frame).unwrap();
        for pkt in packets {
            segment_manager.process_packet(pkt, "monitor:0").await.unwrap();
        }
    }

    segment_manager.finalize_current_segment().await.unwrap();

    // Verify segment was written
    let segments = db.get_all_segments().await.unwrap();
    assert_eq!(segments.len(), 1);
    assert!(Path::new(&segments[0].path).exists());
    assert!(segments[0].is_complete);

    // Verify we can unwrap the segment key
    let seg_key = key_manager.unwrap_segment_key(&segments[0].wrapped_key).await.unwrap();
    assert_eq!(seg_key.len(), 32);
}

// tests/integration/retention_test.rs

#[tokio::test]
async fn test_retention_prunes_old_segments() {
    let db = Db::new_in_memory().await.unwrap();

    // Insert 10 segments at synthetic old timestamps
    for i in 0..10 {
        let ts = utc_now_ms() - (i as i64 * 3600 * 1000);  // i hours ago
        db.insert_segment(test_segment_at(ts)).await.unwrap();
    }

    let config = BufferConfig { max_duration_hours: 3.0, max_disk_gb: 100.0, ..Default::default() };
    let cleaner = RetentionCleaner::new(config, Arc::new(db.clone()), test_dir());

    cleaner.run_once().await.unwrap();

    let remaining = db.get_all_segments().await.unwrap();
    // Only segments from < 3 hours ago should remain
    assert!(remaining.len() <= 3);
}

18.4 Platform Test Matrix
Feature	Windows 10/11	macOS 13+	macOS 12	Linux (X11)	Linux (Wayland)
Screen capture	✅ DXGI	✅ SCKit	⚠️ CGDisplay	✅ XGetImage	⚠️ Portal/PipeWire
Input capture	✅ rdev	✅ rdev	✅ rdev	✅ rdev	⚠️ Limited
Window tracking	✅ Win32	✅ NSWorkspace	✅ NSWorkspace	✅ X11	⚠️ DBus/Portal
OS Keychain	✅ CredMgr	✅ Keychain	✅ Keychain	✅ libsecret	✅ libsecret
Autostart	✅ Registry	✅ LaunchAgent	✅ LaunchAgent	✅ .desktop	✅ .desktop
System tray	✅	✅	✅	✅	⚠️ DE-dependent
19. Build, Packaging & Installation
19.1 Makefile

Makefile

.PHONY: all build build-frontend test test-unit test-integration lint clean install dev release

# Dev mode
dev:
	cargo tauri dev

# Production build
build: build-frontend
	cargo tauri build

# Frontend only
build-frontend:
	cd .. && npm run build

# Tests
test: test-unit test-integration

test-unit:
	cargo test --manifest-path src-tauri/Cargo.toml -- --test-threads=4

test-integration:
	cargo test --manifest-path src-tauri/Cargo.toml \
	  --test integration -- --test-threads=1

# Lint
lint:
	cargo clippy --manifest-path src-tauri/Cargo.toml -- -D warnings
	cd .. && npx svelte-check

# Clean
clean:
	cargo clean --manifest-path src-tauri/Cargo.toml
	rm -rf ../node_modules/.vite

# Release builds (all platforms)
release:
	cargo tauri build --target x86_64-apple-darwin
	cargo tauri build --target aarch64-apple-darwin
	cargo tauri build --target x86_64-pc-windows-msvc
	cargo tauri build --target x86_64-unknown-linux-gnu

# Install locally (macOS)
install:
	cp -r target/release/bundle/macos/BlackBoxLogger.app /Applications/
	@echo "✅ BlackBox Logger installed to /Applications"

19.2 Tauri Build Config

JSON

// src-tauri/tauri.conf.json

{
  "package": {
    "productName": "BlackBox Logger",
    "version": "1.0.0"
  },
  "build": {
    "beforeBuildCommand": "npm run build",
    "beforeDevCommand": "npm run dev",
    "devPath": "http://localhost:5173",
    "distDir": "../dist"
  },
  "tauri": {
    "bundle": {
      "active": true,
      "identifier": "com.blackboxlogger.app",
      "icon": ["icons/32x32.png", "icons/128x128.png", "icons/128x128@2x.png", "icons/icon.icns", "icons/icon.ico"],
      "macOS": {
        "entitlements": "./entitlements.plist",
        "exceptionDomain": "",
        "signingIdentity": null,
        "providerShortName": null
      },
      "windows": {
        "certificateThumbprint": null,
        "digestAlgorithm": "sha256",
        "timestampUrl": ""
      }
    },
    "allowlist": {
      "all": false,
      "dialog": { "open": true, "save": true },
      "notification": { "all": true },
      "window": { "close": true, "hide": true, "show": true, "minimize": true, "setTitle": true }
    },
    "windows": [
      {
        "fullscreen": false,
        "resizable": true,
        "title": "BlackBox Logger",
        "width": 1200,
        "height": 800,
        "minWidth": 900,
        "minHeight": 600,
        "decorations": true,
        "transparent": false,
        "alwaysOnTop": false
      }
    ]
  }
}

19.3 macOS Entitlements

XML

<!-- src-tauri/entitlements.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- Screen Recording permission (required for ScreenCaptureKit) -->
    <key>com.apple.security.screen-capture</key>
    <true/>
    <!-- Accessibility (for input monitoring; may not be needed with SCKit) -->
    <key>com.apple.security.accessibility</key>
    <true/>
    <!-- Keychain access for master key storage -->
    <key>keychain-access-groups</key>
    <array>
        <string>$(AppIdentifierPrefix)com.blackboxlogger.app</string>
    </array>
</dict>
</plist>

19.4 Installation Scripts

Bash

#!/bin/bash
# scripts/install.sh — macOS/Linux installer

set -e

BLACKBOX_HOME="$HOME/.blackbox"
mkdir -p "$BLACKBOX_HOME/segments" "$BLACKBOX_HOME/thumbs" "$BLACKBOX_HOME/logs"

OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
[ "$ARCH" = "x86_64" ] && ARCH="amd64"
[ "$ARCH" = "aarch64" ] || [ "$ARCH" = "arm64" ] && ARCH="arm64"

echo "📦 Downloading BlackBox Logger for ${OS}-${ARCH}..."
RELEASE_URL="https://github.com/yourorg/blackbox-logger/releases/latest/download"
BINARY="blackbox-logger-${OS}-${ARCH}"
curl -L "${RELEASE_URL}/${BINARY}" -o /tmp/blackbox-logger
chmod +x /tmp/blackbox-logger

if [ "$OS" = "darwin" ]; then
    echo "📦 Installing BlackBox Logger.app..."
    cp -r /tmp/BlackBoxLogger.app /Applications/
    echo "🔐 Requesting Screen Recording permission..."
    /Applications/BlackBoxLogger.app/Contents/MacOS/BlackBoxLogger --request-permissions
elif [ "$OS" = "linux" ]; then
    sudo cp /tmp/blackbox-logger /usr/local/bin/blackbox-logger
    # Install .desktop file for autostart support
    mkdir -p ~/.local/share/applications
    cat > ~/.local/share/applications/blackbox-logger.desktop << EOF
[Desktop Entry]
Name=BlackBox Logger
Exec=/usr/local/bin/blackbox-logger
Icon=blackbox-logger
Type=Application
Categories=Utility;
EOF
fi

echo ""
echo "✅ BlackBox Logger installed successfully!"
echo ""
echo "First run:"
echo "  • You will be prompted to set a vault password"
echo "  • Grant Screen Recording permission when prompted (macOS)"
echo "  • The app will start minimized to the system tray"
echo ""
echo "Start BlackBox Logger now? [y/N]"
read -r answer
if [ "$answer" = "y" ] || [ "$answer" = "Y" ]; then
    if [ "$OS" = "darwin" ]; then
        open /Applications/BlackBoxLogger.app
    else
        blackbox-logger &
    fi
fi

PowerShell

# scripts/install.ps1 — Windows installer

param(
    [string]$InstallDir = "$env:LOCALAPPDATA\BlackBoxLogger"
)

$ErrorActionPreference = "Stop"

Write-Host "📦 Installing BlackBox Logger..." -ForegroundColor Cyan
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.blackbox\segments" | Out-Null
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.blackbox\thumbs" | Out-Null
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.blackbox\logs" | Out-Null

$ReleaseUrl = "https://github.com/yourorg/blackbox-logger/releases/latest/download"
$Installer = "$env:TEMP\BlackBoxLogger-setup.msi"

Write-Host "⬇️  Downloading installer..."
Invoke-WebRequest -Uri "$ReleaseUrl/BlackBoxLogger-x64-setup
