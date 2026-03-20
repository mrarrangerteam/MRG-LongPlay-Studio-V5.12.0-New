# LongPlay Studio V5.12.0 — CLAUDE.md

## Key Files
- **gui.py** — Standalone monolith (LongPlayStudioV4, 13k LOC). Run directly: `python gui.py`
- **gui/main.py** — Refactored package version (7k LOC). Used by: `from gui import LongPlayStudioV4`
- **gui/__init__.py** — Re-exports from gui/main.py and gui/ subpackages
- **modules/master/ui_panel.py** — Mastering panel (MasterPanel, MetersPanel, all widgets, 6k+ LOC)
- **modules/master/chain.py** — Audio processing chain (MasterChain, real DSP pipeline)
- **modules/master/maximizer.py** — IRC Maximizer (gain push, ceiling, IRC modes 1-5)
- **modules/master/equalizer.py** — 8-band parametric EQ
- **modules/master/dynamics.py** — Compressor (single + multiband)
- **modules/master/imager.py** — Stereo width (multiband)
- **modules/master/loudness.py** — LUFS/True Peak measurement (ITU-R BS.1770-4)
- **modules/master/limiter.py** — Look-ahead True Peak brickwall limiter
- **modules/master/ai_assist.py** — AI recommendation engine
- **modules/master/genre_profiles.py** — Genre presets, platform targets, IRC modes, tone presets
- **modules/master/loudness_report.py** — LUFS compliance reporting (per-platform pass/fail)
- **gui/widgets/** — Reusable QPainter widgets (rotary knob, vectorscope, transfer curve, etc.)
- **rust/crates/longplay-rt/** — Real-time audio engine (cpal + lock-free DSP)
- **rust/crates/longplay-rt/src/params.rs** — AtomicF32 parameter store (Python -> audio thread)
- **rust/crates/longplay-rt/src/stream.rs** — cpal audio callback + DSP chain per-block
- **rust/crates/longplay-python/src/rt_engine.rs** — PyRtEngine PyO3 bindings

## Dual Entry Points (IMPORTANT)
- `python gui.py` — runs the standalone monolith (used by launch.sh)
- `from gui import LongPlayStudioV4` — imports the refactored version (gui/main.py via gui/__init__.py)
- **Both define LongPlayStudioV4 but they are separate implementations**
- gui/main.py has Phase 2/4 features (undo, themes, text clips, speed ramp, pipeline)
- gui.py has all helper classes inline; gui/main.py imports from gui/ subpackages
- **When fixing bugs, apply to BOTH files if the code exists in both**

## NEVER Do
- **NEVER delete gui.py or gui/main.py** — both are needed (standalone vs package mode)
- **NEVER delete or replace existing working code** — extend it
- **NEVER use fake/random data for meters** — always use real audio samples from chain callbacks
- **NEVER block the audio thread** — no malloc, no file I/O, no mutex, no logging in RT callback
- **NEVER recommend DSP designs without engineering justification** — no hand-waving about "analog warmth" without mechanism
- **NEVER skip True Peak compliance** — all output must pass ITU-R BS.1770-4

## Architecture
- **Hybrid** (V5.10+): Offline rendering + Real-time preview via `longplay-rt` (Rust + cpal)
- **Offline**: chain.py renders full audio for export (Export-Parity)
- **Real-time**: `longplay-rt` plays audio through DSP with lock-free atomic parameters (knob -> instant)
- **Playback**: PyRtEngine (cpal) for mastering preview; QMediaPlayer for timeline playback
- **Meter data**: RT engine -> crossbeam channel -> Python poll 30Hz; chain._send_meter() for offline
- **Signal flow**: Input -> EQ -> Dynamics -> Imager -> Maximizer -> Loudness Norm -> True Peak Limit -> Output
- **RT signal flow**: File -> Memory -> [EQ -> Imager -> Maximizer -> Limiter] -> cpal output (512 samples/block)
- **Audio Thread is Sacred**: Lock-free queues (SPSC ring buffers) for thread communication, pre-allocated memory pools, atomic variables for state flags
- **DAG processing**: Pull model audio graph, fixed block size (512), graph compiled to avoid per-buffer traversal
- **Processor/Editor separation**: Audio processing on RT thread, GUI on main thread, communicate via lock-free atomics

## Tech Stack
- Python 3.14, PyQt6, NumPy, SciPy, pedalboard, pyloudnorm, soundfile
- Rust backend (longplay-dsp, longplay-chain, etc.) compiled to .so via PyO3
- QPainter for all custom widgets (meters, knobs, curves, vectorscope)
- cpal (Rust) for real-time audio I/O, crossbeam for lock-free channels

## Skills Reference
Each `.skill` file in `Skill/` is a ZIP archive containing a SKILL.md router and detailed reference docs.

| Skill File | Description |
|---|---|
| **mrarranger-music-visual.skill** | Music production, Suno AI song creation, visual production, design, high-fidelity DSP engineering (6 skills) |
| **mrarranger-creative-dev.skill** | Mobile dev, game dev, audio plugin dev (VST3/AU/CLAP/JUCE), DAW engine architecture, video editor architecture (5 skills) |
| **mrarranger-dev-tools.skill** | Code master, debug master, CLI tools, MCP builder, technical docs, agent browser, superpowers, vibe code guardian, code review graph (9 skills) |
| **mrarranger-dev-fullstack.skill** | Auth, API master, Supabase, CMS, bridge, React/Next.js, frontend design, UI system, web design, Remotion, SEO (12 skills) |
| **mrarranger-python-backend.skill** | Python backend (Django/FastAPI/Flask), database master (PostgreSQL/MongoDB/Redis), payment systems, data science, accessibility, legal compliance, message queues (7 skills) |
| **mrarranger-devops-ai.skill** | DevOps/cloud, monitoring, webapp testing, data engineer, Web3, AI image gen, AI/ML engineer, AI office (8 skills) |
| **mrarranger-ux-ui-agent.skill** | UX/UI agent with 5-phase loop, design system intelligence, WCAG 2.2 audit, Nielsen heuristics, 60+ component specs, multi-stack code gen |
| **mrarranger-creator-suite.skill** | Creator suite combining UX/UI + video editor (CapCut clone) + graphic design (Canva clone) + clone architecture |
| **openclaw-full-suite.skill** | OpenClaw platform: agents, automation, channels (WhatsApp/Telegram/Slack/Discord), ecosystem, gateway, MCP, office, skills dev (8 skills) |
| **cursor-global-rules.md** | Global Cursor IDE rules: 95+ skill domains, SOLID/Clean Code, React/Next.js/Supabase patterns |

## DSP Engineering Rules
Extracted from `mrarranger-music-visual.skill` (high-fidelity-dsp-engineer reference) and `mrarranger-creative-dev.skill` (audio-plugin-developer, daw-engine-architect references).

### Signal Flow (Mastering Chain Order)
```
Input -> EQ (8-band parametric) -> Dynamics (compressor) -> Imager (stereo width)
     -> Maximizer (IRC modes) -> Loudness Normalization -> True Peak Limiter -> Output
```

### True Peak Compliance (ITU-R BS.1770-4)
- All output MUST comply with platform True Peak targets
- Compliance check: `true_peak_dbtp <= target_tp + 0.1` (0.1 dB tolerance)
- LUFS compliance: `abs(integrated_lufs - target_lufs) <= 1.0 LU`
- Loudness report generates per-platform pass/fail (see `modules/master/loudness_report.py`)

### Look-ahead Limiter Requirements
- IRC 1: 10ms lookahead (transparent peak limiter, minimal coloration)
- IRC 2: 5ms lookahead (adaptive release, program-dependent)
- IRC 3: 5ms lookahead (multi-band 4-band crossover at 120/1k/8k Hz) — MOST POPULAR
- IRC 4/5: Variable lookahead for advanced modes
- Brickwall True Peak limiter is the LAST stage before output

### IRC Mode Specifications (matching iZotope Ozone 12)
- **IRC 1** — Transparent: Simple look-ahead brickwall, no spectral weighting (acoustic/jazz)
- **IRC 2** — Adaptive: Dual-envelope follower, program-dependent release (all-purpose)
- **IRC 3** — Multi-band: 4-band crossover (120/1k/8k Hz), independent limiter per band (default)
  - Sub-modes: Pumping (fast release, EDM), Balanced (natural, default), Crisp (transient-preserving), Clipping
- **IRC 4** — Spectral: FFT-based spectral limiting for maximum transparency
  - Sub-modes: Modern (tight punch), Vintage (warm saturation), Aggressive (loud)
- **IRC 5** — Maximizer: Maximum loudness, designed for competitive levels
- **IRC LL** — Low Latency: Minimal lookahead for live/monitoring use

### DSP Design Principles
1. **Start from the audible goal, not the algorithm** — translate "fatter", "wider", "tighter" into measurable design concerns
2. **Separate signal path from control path** — audio-rate path, envelope/control-rate path, state smoothing, lookahead, gain computer
3. **Account for aliasing** — oversampling needed for nonlinear processing (saturation, waveshaping)
4. **Automation smoothing** — no zipper noise, parameter interpolation required for time-varying systems
5. **Sample-rate independence** — coefficients must adapt to sample rate changes
6. **Validation by both measurement AND listening** — ABX/null tests plus metering

### Processor-Specific Rules
- **EQ/Filters**: Address topology choice, phase implications, coefficient update strategy, stability, sample-rate compensation
- **Compressor/Limiter/Dynamics**: Address detector type (peak/RMS/loudness-weighted), attack/release law, gain computer, lookahead, stereo linking, overshoot
- **Saturation/Nonlinear**: Address transfer curve, harmonic intent, gain staging, oversampling, anti-alias filtering, DC offset
- **Delay/Reverb/Modulation**: Address interpolation quality, diffusion/feedback stability, freeze/tails/bypass handling
- **FFT/Spectral**: Address window choice, hop size, overlap, latency, leakage/temporal smearing, reconstruction quality

## Code Quality Standards
Extracted from `mrarranger-dev-tools.skill`, `mrarranger-dev-fullstack.skill`, and `mrarranger-python-backend.skill`.

### Error Handling Patterns
- Always use try-catch/try-except with specific exception types
- Create custom exception classes for domain-specific errors
- Register global exception handlers for consistent error response formats
- Return proper error messages and log errors with context
- Never swallow exceptions silently

### Thread Safety Rules (Critical for Audio)
- **Audio thread**: NEVER malloc, file I/O, mutex lock, system calls, string ops, or disk logging
- **Lock-free communication**: Use SPSC ring buffers (crossbeam in Rust, lock-free queues in Python)
- **Atomic parameters**: AtomicF32 for knob -> audio thread parameter updates
- **Pre-allocated memory pools**: All buffers allocated at prepareToPlay/init, not during processBlock
- **Meter data**: RT engine writes to lock-free channel, Python polls at 30Hz

### NumPy/SciPy Best Practices
- Use vectorized operations, avoid Python loops for DSP
- Pre-allocate arrays with known shapes
- Use `scipy.signal` for filter design (butter, iirdesign, sosfilt)
- Use `numpy.fft` for spectral processing
- Prefer `float32` for audio buffers (matches hardware), `float64` for coefficient computation

### PyQt6 Thread Rules
- GUI updates ONLY on the main thread
- Use `QTimer` for periodic meter polling (30Hz)
- Use signals/slots for cross-thread communication
- Never call widget methods from the audio thread
- Background processing via `QThread` or `concurrent.futures`

### Code Structure
- Service Layer pattern: routes handle HTTP, services handle business logic, repositories handle data
- Dependency injection for testability
- Feature-based file structure (not type-based)
- Conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`

## Audio Production Rules
Extracted from `mrarranger-music-visual.skill` and `mrarranger-creator-suite.skill`.

### Mastering Chain Order
```
1. EQ (corrective + tonal)
2. Dynamics (compression — single-band or multiband)
3. Stereo Imager (width per frequency band, mono bass below cutoff)
4. Maximizer/Limiter (IRC modes, gain push + ceiling)
5. Loudness Normalization (to platform target)
6. True Peak Brickwall Limiter (final safety, last in chain)
```

### LUFS Targets Per Platform
| Platform | Target LUFS | True Peak (dBTP) |
|---|---|---|
| YouTube | -14.0 | -1.0 |
| Spotify | -14.0 | -1.0 |
| Apple Music | -16.0 | -1.0 |
| Amazon Music | -14.0 | -2.0 |
| Tidal | -14.0 | -1.0 |
| Deezer | -15.0 | -1.0 |
| Podcast | -16.0 | -1.0 |
| Broadcast (EBU R128) | -23.0 | -1.0 |
| CD / Digital Release | -9.0 | 0.0 |
| Vinyl | -12.0 | -0.5 |

### AI Master Workflow (Suno AI -> LongPlay Studio)
1. **Research First** — always research artist/song DNA before production (never guess)
2. **7 DNA Categories**: Harmony, Melodic, Rhythmic, Production, Lyrical, Structural, Emotional
3. **Style Prompt <= 1000 chars** (Suno limit, strictly enforced)
4. **No artist names** in Style Prompts (copyright compliance)
5. **Thai Prosody** for Thai songs: tone-melody mapping (Mid->stable, Low->descending, Falling->high-to-low, High->rising, Rising->low-to-high)
6. **Output format**: 3 Blocks (Lyrics + Style Prompt + Title) for vocal songs; 2 Blocks for instrumentals
7. **QA Pipeline**: Prosody check, format validation, character count, no artist names

### Mastering Presets (15 One-Click)
- Designed for: Dynamics + Imager + Maximizer ONLY (no EQ — Suno AI tone is good)
- Categories: Transparent/Reference, Streaming Optimized, genre-specific (EDM, Hip-Hop, etc.)
- Each preset targets specific IRC mode + gain staging appropriate for genre

## Development Loop (RALP)
1. **R**ead — Read existing code before modifying
2. **A**dd — Add new code, extend existing
3. **L**int — Test with `python3 -c 'from gui import LongPlayStudioV4; print("OK")'`
4. **P**ush — Commit with `feat(STORY-ID): description`
