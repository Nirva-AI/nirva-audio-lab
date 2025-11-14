# 🚀 Quick Start Guide

## Overview

This repo contains all audio ML libraries for Nirva's on-device processing pipeline.

---

## 📁 Structure

```
audio-research/
├── vendor/              # Open-source libraries (with git history)
├── experiments/         # Your research experiments
└── docs/                # Documentation
```

---

## ⚡ Quick Commands

### Build FluidAudio

```bash
cd vendor/FluidAudio
swift build -c release

# Test transcription
swift run -c release fluidaudio transcribe test.wav

# Test diarization
swift run -c release fluidaudio diarization-benchmark \
  --single-file ES2004a
```

### Test CAM++ Voiceprint

```bash
cd vendor/3D-Speaker

# Compare two audio files
python3 scripts/test_speaker_verification.py \
  --audio1 voiceprint.wav \
  --audio2 test.wav \
  --threshold 0.5
```

### Convert Whisper Models

```bash
cd vendor/whisperkittools
pip install -e .

whisperkit-generate-model \
  --model-version large-v3 \
  --output-dir models/
```

---

## 🔄 Working with Vendor Repos

### Pull upstream updates

```bash
cd vendor/FluidAudio
git fetch origin
git merge origin/main

# Commit the update
cd ../..
git add vendor/FluidAudio
git commit -m "Update FluidAudio to latest upstream"
```

### Check local modifications

```bash
cd vendor/3D-Speaker
git log origin/main..HEAD  # Show your local commits
git diff origin/main        # Show all changes
```

### Make local changes

```bash
cd vendor/3D-Speaker
# Edit files
git commit -am "Add new conversion script"

# Update audio-research
cd ../..
git add vendor/3D-Speaker
git commit -m "Update 3D-Speaker with new conversion script"
```

---

## 📚 Documentation

| Document | Location | Purpose |
|----------|----------|---------|
| **Architecture** | `docs/NIRVA_AUDIO_ARCHITECTURE.md` | Complete system design |
| **Speaker Verification** | `vendor/3D-Speaker/SPEAKER_VERIFICATION_COMPLETE.md` | CAM++ integration guide |
| **Library Comparison** | `vendor/3D-Speaker/LIBRARY_COMPARISON.md` | WhisperKit vs FluidAudio |
| **FluidAudio Guide** | `vendor/FluidAudio/CLAUDE.md` | Development guide |

---

## 🎯 Integration Tasks

### Current: CAM++ Speaker Verification ✅

```bash
# Model is ready
vendor/3D-Speaker/campplus_cn_en_common.mlpackage  # 14MB, 192-dim

# Next: Copy to nirva_app
cp -r vendor/3D-Speaker/campplus_cn_en_common.mlpackage \
      ../nirva_app/ios/Runner/Resources/
```

### Next: iOS Integration

1. Create Swift bridge: `nirva_app/ios/Runner/CAMPlusVoiceprintBridge.swift`
2. Create Flutter service: `nirva_app/lib/services/voiceprint_verification_service.dart`
3. Build dev screen with model switcher
4. Test on real recordings

---

## 🔧 Troubleshooting

### "Module not found" in Python

```bash
# Make sure you're in the right directory
cd vendor/3D-Speaker

# Install dependencies
pip install torch coremltools numpy scikit-learn scipy torchaudio

# Add speakerlab to Python path
export PYTHONPATH=$PYTHONPATH:$(pwd)
```

### Swift build fails

```bash
cd vendor/FluidAudio

# Clean build
swift package clean

# Rebuild
swift build -c release
```

### Can't pull from upstream

```bash
cd vendor/FluidAudio

# Check remote
git remote -v

# If origin is missing
git remote add origin https://github.com/argmaxinc/FluidAudio.git
git fetch origin
```

---

## 📊 Model Sizes

| Model | Size | Location |
|-------|------|----------|
| CAM++ | 14 MB | `vendor/3D-Speaker/campplus_cn_en_common.mlpackage` |
| Parakeet TDT | ~1 GB | FluidAudio auto-downloads |
| WeSpeaker | ~50 MB | FluidAudio auto-downloads |

---

## 🎓 Learning Resources

- **CAM++ Paper**: https://arxiv.org/abs/2303.00332
- **FluidAudio**: https://github.com/argmaxinc/FluidAudio
- **WhisperKit**: https://github.com/argmaxinc/WhisperKit
- **3D-Speaker**: https://github.com/alibaba-damo-academy/3D-Speaker

---

**Ready to integrate? Start with `docs/NIRVA_AUDIO_ARCHITECTURE.md`**
