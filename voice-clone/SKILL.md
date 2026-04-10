---
name: voice-clone
preamble-tier: 3
version: 1.0.0
description: |
  Clone Howard's voice to generate Chinese/English speech using VoxCPM2 (2B params, 30 languages).
  Runs on Mac mini M2 (193) via SSH. Ultimate Cloning mode: reference audio + transcript → exact voice replica.
  Use when asked to "generate speech", "voice clone", "say this in my voice", "克隆声音", "用我的声音说".
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - SSH
---

## When to Use

✅ **USE this skill when:**
- "用我的声音说..." / "say this in my voice"
- "帮我生成一段语音" / "generate speech"
- "voice clone" / "克隆声音"
- Any TTS request in Howard's voice (Chinese or English)

❌ **DON'T use when:**
- Just playing existing audio files
- System notifications (use standard TTS tool instead)
- Non-Howard voice requests

## Prerequisites

- VoxCPM2 model downloaded on 193
- Python environment on 193
- Reference audio with accurate transcript

## Step 0: Environment Setup

Set variables:

```bash
HOST="100.112.125.107"  # 193 Tailscale IP
USER="dda"
PYTHON="/Users/dda/services/fish-speech/.venv/bin/python3"
REF_AUDIO="/Users/dda/howard-voice-data/recordings/chinese/chinese_001.wav"
REF_TEXT="你好，这是我的声音测试"
SCRIPT="/tmp/voxcpm_gen.py"
OUTPUT="/tmp/howard_demo/voxcpm_out.wav"
```

## Step 1: Generate Speech

```bash
ssh ${USER}@${HOST} "PATH=/opt/homebrew/bin:\$PATH ${PYTHON} ${SCRIPT} '${TEXT}' '${OUTPUT}'"
```

The script uses VoxCPM2 Ultimate Cloning:
- `prompt_wav_path` = reference audio (for continuation)
- `prompt_text` = reference transcript (must be accurate!)
- `reference_wav_path` = same reference audio (for voice cloning)

## Step 2: Copy and Deliver

```bash
scp ${USER}@${HOST}:${OUTPUT} /tmp/voxcpm_result.wav
# Send to user via messaging tool or direct audio
```

## Architecture

| Component | Details |
|-----------|---------|
| Model | VoxCPM2 (2B params, openbmb/VoxCPM2) |
| Mode | Ultimate Cloning (prompt + reference) |
| Languages | 30 (auto-detect from text) |
| Output | 48kHz WAV |
| RTF | ~0.2 (5x faster than real-time) |
| First load | ~15 seconds |
| Per-sentence | ~3-5 seconds |
| GPU | Apple M2 MPS |
| Source | https://github.com/dddabtc/voxcpm |

## Reference Audio

The quality of cloning depends heavily on:

1. **Reference audio clarity** — low noise, clear speech
2. **Transcript accuracy** — must match what's actually said
3. **Language match** — Chinese reference for Chinese text works best

Current reference: `chinese_001.wav` (2.2s, "你好，这是我的声音测试")

## Notes

- `CUDA_VISIBLE_DEVICES=''` forces MPS (model defaults to CUDA)
- `load_denoiser=False` avoids ModelScope dependency issues
- Script lives at `/tmp/voxcpm_gen.py` on 193
- Model cache: `~/voxcpm_models/`
- Source: `~/voxcpm/src/`
