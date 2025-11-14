# 🎙️ Nirva Audio Lab

**Consolidated repository for audio ML research, experiments, and Streamlit processing app.**

This repo combines:
- **Streamlit App**: Python-based audio analysis and transcription UI (`app.py`)
- **Vendor Libraries**: All audio ML libraries for on-device processing (`vendor/`)
- **Experiments**: Research experiments and prototypes (`experiments/`)
- **Documentation**: Architecture and integration guides (`docs/`)

---

## 📁 Repository Structure

```
nirva-audio-lab/
├── app.py                  # Streamlit audio processing app
├── audio_utils.py          # Audio processing utilities
├── transcription.py        # Transcription logic
├── requirements.txt        # Python dependencies
│
├── vendor/                 # Vendored ML libraries (with git history)
│   ├── FluidAudio/         # Complete audio pipeline (ASR, VAD, Diarization)
│   ├── WhisperKit/         # On-device Whisper ASR for iOS
│   ├── whisperkittools/    # Model conversion and optimization
│   ├── 3D-Speaker/         # CAM++ speaker verification (⚠️ local changes)
│   └── pyannote-audio/     # Speaker diarization research
│
├── experiments/            # Research experiments
├── docs/                   # Documentation
│   └── NIRVA_AUDIO_ARCHITECTURE.md
└── QUICK_START.md         # Quick reference guide
```

---

## ✨ Streamlit App Features

- **✂️ Silence Detection and Trimming**
  - Automatically detect and remove silence segments
  - Visual timeline representation of audio content
  - Adjustable silence threshold and minimum length
  - Download trimmed audio files

- **🔄 Audio Segmentation**
  - Split audio into meaningful segments
  - Intelligent boundary detection
  - Preserve context between segments

- **🗣️ Advanced Transcription**
  - Speaker identification and diarization
  - High-accuracy speech-to-text conversion
  - Support for multiple audio formats (WAV, MP3, M4A)

- **😊 Emotion Detection**
  - Analyze speaker emotions
  - Identify emotional patterns and changes
  - Emotional context annotation

- **🎵 Ambient Sound Classification**
  - Detect and classify background sounds
  - Environmental context analysis
  - Noise type identification

## 🚀 Getting Started

### Prerequisites

- Python 3.10 or higher
- Conda package manager

### Installation

1. Create and activate a conda environment:
```bash
conda create -n nirva python=3.10
conda activate nirva
```

2. Install required packages in the following order:
```bash
# Basic scientific packages
conda install numpy scipy pandas

# PyTorch and torchaudio
conda install pytorch==1.13.1 torchaudio==0.13.1 -c pytorch

# Audio processing libraries
conda install -c conda-forge librosa soundfile
pip install pydub

# Transformers and dependencies
pip install transformers

# Whisper
pip install openai-whisper

# Streamlit
pip install streamlit

# Pyannote audio
pip install pyannote.audio==2.1.1
```

### Usage

1. Start the application:
```bash
streamlit run app.py
```

2. Access the web interface at `http://localhost:8501`

3. Upload audio files and process them using the available features:
   - Select files to process
   - Adjust silence detection parameters if needed
   - Choose between silence trimming and full processing
   - Download processed results

## 🛠️ Technical Details

- **Frontend**: Streamlit
- **Audio Processing**: PyTorch, torchaudio, librosa
- **Speech Recognition**: Whisper
- **Speaker Diarization**: pyannote.audio
- **Visualization**: Matplotlib, Streamlit components

## 📊 Processing Pipeline

1. **File Upload**
   - Support for multiple file uploads
   - Automatic format detection
   - Initial audio analysis

2. **Silence Processing**
   - Configurable silence detection
   - Visual timeline generation
   - Non-destructive trimming

3. **Audio Analysis**
   - Speaker diarization
   - Transcription
   - Emotion and ambient sound analysis

4. **Results Presentation**
   - Interactive visualizations
   - Downloadable processed files
   - Detailed analytics

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📝 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🙏 Acknowledgments

- OpenAI Whisper for speech recognition
- Pyannote.audio for speaker diarization
- Streamlit for the web interface

---

## 📦 Vendor Libraries

| Library | Purpose | Status | Source |
|---------|---------|--------|--------|
| **FluidAudio** | ASR + Diarization pipeline | ✅ Unmodified | [argmaxinc/FluidAudio](https://github.com/argmaxinc/FluidAudio) |
| **WhisperKit** | On-device Whisper ASR | ✅ Unmodified | [argmaxinc/WhisperKit](https://github.com/argmaxinc/WhisperKit) |
| **whisperkittools** | Model optimization | ✅ Unmodified | [argmaxinc/whisperkittools](https://github.com/argmaxinc/whisperkittools) |
| **3D-Speaker** | CAM++ voiceprint verification | ⚠️ **Local changes** | [alibaba-damo/3D-Speaker](https://github.com/alibaba-damo-academy/3D-Speaker) |
| **pyannote-audio** | Diarization research | ⚠️ **Local changes** | [pyannote/pyannote-audio](https://github.com/pyannote/pyannote-audio) |

### 3D-Speaker Modifications

**Added files:**
- `scripts/convert_campplus_to_coreml.py` - PyTorch → CoreML conversion
- `scripts/test_speaker_verification.py` - Verification testing script
- `campplus_cn_en_common.mlpackage/` - CoreML model (14MB, 192-dim embeddings)
- `SPEAKER_VERIFICATION_COMPLETE.md` - Integration guide

### Working with Vendor Libraries

Each vendor library maintains its own git history. You can:

```bash
# Make local changes
cd vendor/3D-Speaker
git commit -am "Add new feature"

# Pull upstream updates
cd vendor/FluidAudio
git fetch origin
git merge origin/main

# See what you've changed
cd vendor/3D-Speaker
git log origin/main..HEAD  # Show local commits
git diff origin/main        # Show all changes
```

---

## 📚 Documentation

- **[QUICK_START.md](QUICK_START.md)** - Quick command reference
- **[docs/NIRVA_AUDIO_ARCHITECTURE.md](docs/NIRVA_AUDIO_ARCHITECTURE.md)** - Complete on-device processing pipeline
- **[vendor/3D-Speaker/SPEAKER_VERIFICATION_COMPLETE.md](vendor/3D-Speaker/SPEAKER_VERIFICATION_COMPLETE.md)** - CAM++ integration guide
- **[vendor/FluidAudio/CLAUDE.md](vendor/FluidAudio/CLAUDE.md)** - FluidAudio development guide

---

## 🎯 On-Device Audio Pipeline (nirva_app integration)

The vendor libraries support Nirva's 4-task on-device processing:

1. **Offline ASR** - Necklace stores audio as files
2. **Periodic Processing** - On-device ASR to diarized transcript
3. **Voiceprint Verification** - Compare diarized speakers with user voiceprint, tag as "self"
4. **Dev Screen** - Toggle cloud vs on-device models

See [docs/NIRVA_AUDIO_ARCHITECTURE.md](docs/NIRVA_AUDIO_ARCHITECTURE.md) for complete architecture.

---

## 📊 Model Inventory

### On-Device Models (iOS/macOS)

| Model | Size | Task | Location |
|-------|------|------|----------|
| **CAM++** | 14 MB | Speaker Verification | `vendor/3D-Speaker/campplus_cn_en_common.mlpackage` |
| **Parakeet TDT v3** | ~1 GB | ASR (25 languages) | FluidAudio auto-download |
| **WeSpeaker v2** | ~50 MB | Speaker Embeddings | FluidAudio auto-download |
| **Silero VAD** | ~5 MB | Voice Activity Detection | FluidAudio auto-download |