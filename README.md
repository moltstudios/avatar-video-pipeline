# Avatar Video Pipeline

Autonomous AI video production system that generates character-consistent, lip-synced short-form videos from text scripts. Five AI models orchestrated in sequence: script generation, character-locked image generation, neural TTS, audio-driven avatar animation, multi-track audio mixing, final render. Production-deployed and producing content daily.

This is not a proof of concept. It produces real videos for real social media channels.

https://github.com/user-attachments/assets/bcf4754c-2571-4574-804a-e583b1f64e74

> **How it works** — See the [Pipeline Architecture](#pipeline-architecture) section below. Full case study with sample output in [Results](#results).

## Pipeline Architecture

Script Engine -> Character Lock -> Scene Refs -> TTS Voice -> Avatar Animation -> Assembly -> SFX + Music -> Final Render

**End-to-end time:** ~30-45 min per video, ~4 hours for a batch of 6  
**Cost:** ~$0.02 per scene (Modal A100-80GB)

## The 5 Stages

### 1. Script Generation

AI-generated scripts built on a research-backed content framework:

- **12 hook formulas** (contrarian, stat, listicle, transformation, founder, curiosity gap, etc.)
- **5-beat video structure:** Hook (0-3s), Context (3-8s), Problem (8-15s), Reveal (15-25s), CTA (25-30s)
- **Platform-specific pacing:** 150-170 WPM for 30s TikTok/Reels, 120-140 WPM for 60s Shorts
- **4 content pillars:** Edu-tainment, Entertainment, Demo, Social Proof
- **34,000-word content bible** codifying what works across platforms

Scripts aren't generated freestyle. Each one maps to a specific hook formula, targets a measured word count, and follows the 5-beat structure proven to maximize completion rate.

### 2. Character-Locked Image Generation

The hardest problem in AI video: **keeping a character consistent across every scene.** Our solution:

**Phase 1 - Character Lock (mandatory gate):**
- Generate full-body character sheet (front-facing, neutral background, all defining features visible)
- Human approval required before ANY scene generation begins
- Locked reference stored with manifest - never regenerated mid-project

**Phase 2 - Scene References:**
- Each scene starts from the locked character reference (passed as image input)
- Background baked INTO the reference image (LongCat regenerates full frames, so the reference must contain the desired scene)
- Quality gate per reference: Pixar style? 9:16 portrait? Character matches lock? Correct emotion? No text artifacts?

**Image model:** GPT-Image (OpenAI) for primary generation, Gemini Flash Lite for forensic visual analysis.

### 3. Neural Text-to-Speech

**Model:** Qwen3-TTS with custom "sassy" voice preset

- Conversational tone - sounds like talking to a friend, not reading a script
- Natural cadence with varied speed, pauses, and pitch shifts
- Emotional range matched to content (excitement for reveals, concern for problems)
- Post-processing chain: silence trimming, 0.3s padding, 16kHz mono formatting

**Critical rule:** Audio is generated BEFORE video. LongCat uses audio as a conditioning signal - it drives frame-by-frame mouth movements. You cannot swap audio after rendering without it looking like a dubbed movie.

### 4. Avatar Animation (LongCat-Video-Avatar-1.5)

The core engine. A 22B-parameter audio-to-video model deployed on Modal (A100-80GB).

**What it does:**
- Full-body animation: shoulders, arms, hands, and gestures move naturally with speech
- Audio-driven lip sync: mouth movements frame-by-frame matched to audio input
- Multi-person conversations: `generate_avatar_multi` renders 2+ characters in one frame with independent lip-sync per character
- Style generalization: works with Pixar 3D, anime, anthropomorphic objects (tested with talking car parts, fruit characters)
- Physics: sparks, rotations, particles, prop stability all animate naturally. Fluids/smoke don't work.
- Built-in vocal separation for clean voice extraction

**What it can't do (hard limits, documented from production):**
- No scene transitions within a single generation (one continuous shot)
- No walking/locomotion (upper body only)
- No physical interaction between characters (talking only)
- No exact background preservation (regenerates full frame)
- No real-time generation (~10-20 min per 30s clip)

**Deployment:**
- App: `longcat-avatar` on Modal Labs
- Volume: `longcat-models` (35GB cached weights)
- GPU: A100-80GB (42.1 GB peak VRAM)
- Frame rate: 25fps (8fps native, interpolated)
- Output: 480x832 portrait (auto-selected from 9:16 input via bucket system)

### 5. Audio Post-Production

Multi-track mixing that takes a video from "good" to "viral":

**22-SFX library** across 7 categories:
- **Impacts:** sparkle shimmer, bass drop, cinematic boom, door slam, glass shatter
- **Transitions:** whoosh (fast/medium/slow)
- **Foley:** phone ring, champagne pop, coins, drink pour, evil laugh
- **Weapons:** chainsaw rev (tested in production)
- **Atmosphere:** suspense drone, dramatic sting, heartbeat
- **UI:** notification ding, success ding
- **Vehicles:** car engine rev

**Mixing rules (non-negotiable):**
- Dialogue is king - always loudest in the mix (100%)
- SFX at 60-80% - punchy, doesn't overpower speech
- Background music at 18-22% during silence, **ducks to ~6%** during speech
- One SFX per key moment - never over-layer

**Background music:** Custom default beat (3:25, designed for AI fruit/drama content)

## Multi-Character Production

Tested and proven with dual-character dialogue scenes:

- Both characters render in a single frame with independent lip-sync
- Only the speaker's face moves; the listener stays animated but idle
- Character-to-audio mapping is the #1 mistake - always verify which character speaks first
- `generate_avatar_multi(image, audio_p1, audio_p2)` - P1 maps to left/top, P2 to right/bottom

## Assembly and Export

For videos longer than 3.5 seconds (the per-generation sweet spot at 89 frames):

1. Split script at natural pause points
2. Generate each segment with matching reference image
3. Concatenate via ffmpeg
4. Re-mux with full audio track
5. Mix SFX and background music
6. Export at 9:16 portrait, compressed for platform limits

Mixed assembly interleaves solo cuts (single-character moments) with multi-person renders (conversation scenes) in storyboard order.

## Key Innovations

**Character Lock Protocol** - The #1 quality issue in AI video is character drift between scenes. Mandatory two-phase system: lock the character first (human approval gate), then generate all scenes from that locked reference. Eliminated ~90% of consistency issues.

**Audio-First Pipeline Ordering** - Most pipelines generate video first, then add audio. Ours reverses this: audio drives animation frame-by-frame, so TTS must come before avatar generation. Produces genuine lip sync, not face-swap approximations.

**SFX Placement Strategy** - Every video gets a mental pass: "what SOUND would be here?" SFX at exact action moments - discoveries get sparkles, calls get rings, power tools get revs. This separates flat from viral.

### Lessons From Production (8 Iterations)

| Lesson | Impact |
|--------|--------|
| Character drift without locked refs | #1 quality issue, solved by mandatory lock phase |
| Audio desync compounds across splits | Each segment creates micro-gaps, verify total duration |
| Multi-character voice mapping | P1=left char, P2=right char, always verify speaker order |
| Physics limitations | Sparks yes, rotation yes, smoke no, fluids no |
| Frame count constraint | Must satisfy (n-1) % 4 == 0 or generation fails |
| Expression drift in long segments | Keep under 89 frames (3.5s) for best results |
| Reference image = 80% of output quality | Great ref + mediocre prompt beats mediocre ref + great prompt |
| Anthropomorphic objects work | "Machines with faces" not "faces on machines" |

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|--------|
| Script Engine | Custom framework + LLM | Hook formulas, 5-beat structure, platform rules |
| Image Generation | GPT-Image (OpenAI) | Character-locked scene references |
| Forensic Analysis | Gemini Flash Lite | Visual detail extraction for refs |
| Text-to-Speech | Qwen3-TTS (sassy preset) | Neural voice generation |
| Avatar Animation | LongCat-Video-Avatar-1.5 | Audio-driven lip sync on A100 |
| GPU Compute | Modal Labs (A100-80GB) | Cloud GPU inference |
| Audio Mixing | Custom mix_audio.py + ffmpeg | Multi-track: dialogue + SFX + music |
| SFX Library | 22 synthesized effects | Impacts, transitions, foley, atmosphere |
| Video Assembly | ffmpeg | Concatenation, re-muxing, export |

## Results

### Case Study: UGC-Style Product Video

End-to-end pipeline producing a user-generated content (UGC) style product demonstration. A real product photo and lifestyle reference were fed into the pipeline. The AI character (YUMI) was composited into a car selfie scene holding the product, with audio-driven lip sync and natural gestures.

<table>
  <tr>
    <td align="center">
      <img src="assets/suave-product-reference.jpg" width="250" alt="Lifestyle pose reference"><br>
      <sub><b>1. Lifestyle Reference</b></sub><br>
      <sub>Pose and scene composition target</sub>
    </td>
    <td align="center">
      <img src="assets/suave-lifestyle-input.jpg" width="250" alt="Product reference input"><br>
      <sub><b>2. Product Reference</b></sub><br>
      <sub>Source product photo (white background)</sub>
    </td>
    <td align="center">
      <img src="assets/yumi-suave-ugc.png" width="250" alt="AI character UGC output"><br>
      <sub><b>3. AI Character Output</b></sub><br>
      <sub>YUMI holding product, car selfie style</sub>
    </td>
  </tr>
</table>

The final video features the character speaking to camera with the product, generating a natural UGC-style product endorsement. The entire sequence — image generation, TTS voice, avatar animation, SFX mixing — runs on custom cloud GPU infrastructure (Modal A100-80GB) at approximately $0.02 per scene.

https://github.com/user-attachments/assets/bcf4754c-2571-4574-804a-e583b1f64e74

### Production Metrics

- **Production-deployed** - not a demo, produces content daily
- **Character consistency** across multi-scene videos (solved via lock protocol)
- **Multi-character dialogue** with independent lip-sync per character
- **22-SFX library** with mixing system that matches professional content
- **34,000-word content bible** backing every creative decision
- **Platform-optimized** output for TikTok, Instagram Reels, YouTube Shorts

## License

MIT

---

Built by [Molt Studios](https://github.com/moltstudios)
