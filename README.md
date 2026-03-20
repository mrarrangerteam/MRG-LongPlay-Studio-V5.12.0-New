# MRG LongPlay Studio V5.12.0

**CapCut-class Video Editor + Logic Pro X + iZotope Ozone 12 Mastering**

A professional-grade desktop application for AI-powered music production, video editing, and audio mastering. Built with Python (PyQt6) + Rust (PyO3) for native DSP performance.

## Features

### 🎬 Video Editor (CapCut-style)
- Multi-track timeline with drag & drop
- GIF/Logo overlay (size, position, opacity)
- Video transitions with crossfade
- Batch export mode
- Hardware-accelerated encoding (VideoToolbox, NVENC, AMF, QSV, VAAPI)
- Smart temp directory selection for large exports

### 🎵 AI DJ
- BPM, Energy, Key analysis for smart playlist ordering
- Multiple shuffle algorithms
- Auto-numbering and rename
- Preview crossfade between tracks

### 🎛️ AI Mastering (Ozone 12-quality)
- **8-Band Parametric EQ** with genre presets and tone presets
- **Multiband Dynamics** (3-band compressor with crossover control)
- **Stereo Imager** (M/S processing, multiband, mono bass)
- **Maximizer** with IRC Modes 1-5 + Low Latency (sub-modes: Pumping/Balanced/Crisp/Classic/Modern/Transient)
- **True Peak Limiter** (4x oversampled ISP detection, ITU-R BS.1770-4 compliant)
- **AI Assist** — Genre-aware auto-mastering with 30+ genre profiles
- **Loudness Normalization** to platform targets (YouTube, Spotify, Apple Music, etc.)

### 📊 Metering (Waves WLM Plus-inspired)
- LUFS Momentary / Short-term / Integrated
- True Peak L/R per-channel display
- LRA (Loudness Range)
- Gain Reduction History timeline
- Waves-inspired dark teal UI with LED segments

### 🎨 UI Design
- Waves/SSL/Neve hardware-inspired aesthetic
- Gunmetal chassis, warm amber VU glow, brushed steel knobs
- Dark mode with teal/cyan accents

## Architecture

```
┌─────────────────────────────────────────────┐
│           Layer 3: PyQt6 GUI                │
│  Waves-inspired Mastering Panel + Timeline  │
├─────────────────────────────────────────────┤
│        Layer 2: Python Application          │
│  AI Assist, Video Editor, AI DJ, Hooks      │
├─────────────────────────────────────────────┤
│        Layer 1: Rust Native Engine          │
│  DSP, Analysis, Chain, I/O (via PyO3)       │
│  Fallback: Python (pedalboard + scipy)      │
└─────────────────────────────────────────────┘
```

### Backend Priority
1. **Rust** (PyO3) — Native performance, 10-100x faster
2. **C++** (pybind11) — Optional alternative
3. **Python** (pedalboard + scipy) — Always-available fallback

## Requirements

- **Python** 3.10+ (tested on 3.12, 3.14)
- **FFmpeg** (brew install ffmpeg / apt install ffmpeg)
- **PyQt6** 6.6.0+
- **Rust** 1.70+ (optional, for native backend)

## Installation

```bash
# Clone repository
git clone https://github.com/mrarrangerteam/MRG-LongPlay-Studio-V5.12.0.git
cd MRG-LongPlay-Studio-V5.12.0

# Install Python dependencies
pip3 install -r requirements.txt

# Run the application
python3 gui.py
```

### Optional: Build Rust Native Backend

```bash
# Install maturin
pip3 install maturin

# Build and install Rust module
cd rust
maturin develop --release

# Verify Rust backend is active
python3 -c "import longplay; print('Rust backend available!')"
```

## Project Structure

```
MRG-LongPlay-Studio-V5.12.0/
├── gui.py                    # Main application (PyQt6)
├── ai_dj.py                  # AI DJ playlist engine
├── hook_extractor.py         # Audio hook detection
├── video_prompt_generator.py # AI video prompt generation
├── license_manager.py        # License validation
├── requirements.txt          # Python dependencies
├── modules/
│   └── master/               # AI Mastering Module
│       ├── __init__.py       # Backend auto-detection (Rust→C++→Python)
│       ├── chain.py          # Master chain orchestrator (Python)
│       ├── rust_chain.py     # Rust/C++ backend bridge
│       ├── equalizer.py      # 8-band Parametric EQ
│       ├── dynamics.py       # Multiband dynamics/compressor
│       ├── imager.py         # Stereo imager (M/S)
│       ├── maximizer.py      # Ozone 12-style maximizer
│       ├── limiter.py        # Look-ahead true peak limiter
│       ├── loudness.py       # LUFS measurement
│       ├── analyzer.py       # Audio analysis (spectral, dynamic, stereo)
│       ├── ai_assist.py      # AI recommendation engine
│       ├── genre_profiles.py # 30+ genre profiles, IRC modes
│       └── ui_panel.py       # Waves-inspired mastering GUI
└── rust/                     # Rust native engine
    ├── Cargo.toml            # Workspace config
    └── crates/
        ├── longplay-core/    # Type definitions, conversions
        ├── longplay-dsp/     # EQ, Dynamics, Imager, Maximizer, IRC Limiter
        ├── longplay-chain/   # Master chain orchestrator
        ├── longplay-analysis/# FFT, LUFS, spectral, stereo analysis
        ├── longplay-profiles/# Genre profiles, tone presets
        ├── longplay-ai/      # AI recommendation engine
        ├── longplay-io/      # WAV I/O (hound)
        ├── longplay-python/  # PyO3 bindings
        ├── longplay-cli/     # CLI interface
        ├── longplay-aidj/    # AI DJ (Rust)
        └── longplay-hooks/   # Hook extraction (Rust)
```

## Signal Flow (Mastering Chain)

```
Input → Pre-gain (-3dB) → EQ → Dynamics → Imager → Maximizer → Loudness Norm → True Peak Limit → Output
```

## Supported Platforms

| Platform | Target LUFS | True Peak |
|----------|------------|-----------|
| YouTube | -14.0 | -1.0 dBTP |
| Spotify | -14.0 | -1.0 dBTP |
| Apple Music | -16.0 | -1.0 dBTP |
| Tidal | -14.0 | -1.0 dBTP |
| Amazon Music | -14.0 | -2.0 dBTP |
| SoundCloud | -14.0 | -1.0 dBTP |
| CD | -9.0 | -0.3 dBTP |
| Radio | -23.0 | -1.0 dBTP |

## Changelog

### V5.12.0 (2026-03-20)
- Merged all V5.11.0-RUST (Rust DSP backend via PyO3) and V5.11.1 bug fixes into a single clean release
- **Limiter fix**: Empty array safety check (`os_end > os_start`) prevents crash on zero-length oversampled segments
- **Chain fix**: `band_ceiling = ceiling_linear` — removed incorrect 0.95/0.9/0.85 multipliers that over-limited multiband output
- **GUI fix**: `final_true_peak_limit()` used in `_render_mastered` instead of `np.clip` for proper ISP-safe true peak limiting
- **Meter panels fix**: `_auto_find_audio` and `_auto_feed_spectrum` added for automatic audio source discovery in standalone meter panels
- **Code quality**: Added try/except guard around spectrum chunk data in `_send_meter` for robustness
- **Resonance Suppressor**: Soothe2-style dynamic resonance suppression module included
- **Real-time engine**: Rust cpal-based low-latency preview with lock-free atomic parameters

### V5.11.0-RUST
- Full Rust DSP backend (longplay-dsp, longplay-chain, longplay-rt) compiled via PyO3
- Real-time audio preview engine using cpal + crossbeam lock-free channels
- IRC Maximizer modes 1-5 ported to Rust with SIMD-friendly inner loops

### V5.10.x
- Per-band gain reduction meters for dynamics popup
- Stereo correlation and width meters for imager popup
- LRA (Loudness Range) measurement per ITU-R BS.1770-4
- Spectrum analyzer feed in meter callbacks

## License

MIT License

## Author

**MRARRANGER AI Studio** — 20 years of music production experience, now AI-augmented.
